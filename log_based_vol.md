# Designing the Log-based HDF5 VOL Plugin
As part of the ECP, we are exploring ways to implement the idea of storing write requests in a log-based layout in HDF5 files.
The goal is to achieve fast parallel write performance by avoiding the expensive cost spent on inter-process communication in the MPI collective I/O.

The latest HDF5 development incorporated an abstraction layer named the Virtual Object Layer (VOL) that allows advanced users to customize their I/O operations by supplying an I/O driver as a plug-in library module that can be developed independently from HDF5 and linked together with the native HDF5 libraries.
To implement the log-based data layout, we use the HDF5 VOL as a platform to exercise various ideas and study their performance.
The lessons learned from this exploration can help us understand the current limitations of HDF5 VOL and serve as a guidance for developers for future improvement.

This document serves as a place holder to encourage internal discussions, suggestions, and to show preliminary results for verifying the ideas.

---

## Various Approaches:
* [Make use of HDF5 VDS](#make-use-of-hdf5-vds)
* [A Log-based VOL Plugin](#a-log-based-vol-plugin)
* [Supporting Read Operations](#supporting-read-operations)
* [Other Lessons Learned](#other-lessons-learned)

---

### Make use of HDF5 VDS

HDF5 Virtual Dataset (VDS) is a feature that allows to store a dataset in a layout different from the regular HDF5 dataset, but can still be accessed transparently like a regular dataset.
This lets us to implement the log-based layout on top of an existing HDF5 framework for better portability.
A VDS may be comprised of multiple HDF5 regular datasets underneath, referred as the `target` datasets.
If we used a single VDS to store multiple write requests made by users, then we can achieve the effect of log-based data layout.
Creating a VDS requires users to construct multiple links that map part of its data space to the space of target datasets.

A short description for creating a VDS
* A VDS is created by setting the mappings in the dataset creation property list.
* A mapping can be a portion of the data space.
* Target datasets can be datasets stored in external files or the same file.
* The selection of VDS and target dataset does not need to be the same, but the size must match.
Detailed information about the VDS can be found in https://support.hdfgroup.org/HDF5/docNewFeatures/NewFeaturesVirtualDatasetDocs.html

We are considering the following implementation.
First, we create a virtual dataset for each dataset is created by the user application.
For each write request made by the user application, our VOL defines a mapping in the corresponding virtual dataset that maps the selected space to the data in the log dataset.
Once write data is represented by VDS, reading the dataset can be done via the HDF5 VDS mapping mechanism, so the actual read operations are redirected to the corresponding target dataset.

#### Current Limitations of VDS
* **Lack of MPI-IO support** -- Currently, MPI-IO is not supported for VDS feature.
   Thus, we need to develop in our VOL plugin an interface to the MPI VFL driver.

  + Delay virtual dataset creation** --
    A workaround is to delay the creation of virtual datasets until the file is closed.
    The metadata is kept in memory during and session and gathered to the master rank (rank 0).
    The master rank reopens the file using the POSIX VFL driver and creates all virtual datasets.
  + Open the file in read-only mode
    The easiest way to support reading is to have individual processes open the file using the POSIX VFL driver.
    It requires the file to be opened in read-only mode, so HDF5 will not try to lock the file.
    As a result, read and write modes are not supported.

#### Performance Study of VDS
* Test program [vds.cpp](./vds.cpp) -- it first creates a 1024 x 1024 dataset of type int32 in a contiguous layout, followed by creating a VDS of the same size.
  The n-th column of the virtual dataset maps to the n-th row of the contiguous dataset.
  It reports a performance comparison of a column-wise access to the contiguous and the virtual dataset.
* We expect accessing the virtual dataset to be faster than the contiguous dataset, because the access pattern for the VDS is contiguous in the file space.
  However, we observed a poorer timings of accessing the virtual dataset.
  Further study reveals that the slow performance is caused by repeated file open and close operations.
* Performance results of running the test program on a local server:
  ```
  Create virtual dataset: 80 ms

  Row-wise write on the contiguous dataset: 20 ms
  Column-wise write on the contiguous dataset: 3496 ms
  Column-wise write on the virtual dataset: 22764 ms

  Row-wise read on contiguous dataset: 20 ms
  Column-wise read on the contiguous dataset: 1774 ms
  Column-wise read on the virtual dataset: 22143 ms
  ```
* **Analysis** -- The VDS feature is designed to allow linking datasets across multiple files.
  During the call to `H5Dread()`, the data access selection is first compared against the VDS mappings.
  For each intersection, HDF5 opens the source file, reads the data, and close the file.
  For a VDS with many mappings, the frequent file open and close operations become significant to hurt the performance.
* **Discussions**
  + There is no workaround outside the HDF5.
  + Develop a patch to HDF5 that uses a file cache to mitigate the poor performance.
  + Develop a patch to HDF5 to keep the file open for future access.

---

### A Log-based VOL Plugin
A design document contains the idea of implementing the log-based data layout in an HDF5 VOL plugin is available in [./design_log_base_vol.docx](./design_log_base_vol.docx).
This approach does not make use of HDF5 VDS.
Instead, it implements its own mappings and access mechanism to the data stored in the log layout.


---

### Supporting Read Operations 
* Constructing the metadata table
  To allow efficient searching for write requests in log datasets, we construct a metadata table of the log entries.
  Each entry of the metadata table stores the metadata of a write request. 
  The metadata includes the location of data in the file and where the data belongs to (which dataset, which part).
  Entries are sorted based on increasing order of dataset IDs of the dataset involved.
  + Storing the metadata table as an unlimited 1-dimensional byte dataset
    There is only one metadata table dataset.
    The dataset must be expandable to accommodate future writes.
    HDF5 requires all datasets with unlimited size to be chunked.
  + Accelerate table lookup with a lookup table 
    We create a lookup table containing the offset of the first metadata entry of every dataset.
    It allows us to skip through irrelevant entries.
* Reading from the log entries 
  The VOL traverse the metadata table and look for entries that intersect the selection.
  For every intersection, the VOL retrieve the intersecting part of the data and place it into the correct position in the application buffer.
  + HDF5 selection always assumes canonical order
    Regardless of the order selection is made, HDF5 always reads selected data according to its position in canonical order.
    For example, we cannot create a selection that reads the rows of a 2-D dataset in reverse order.
    The same behavior applies to memory space selections.
    For this reason, we can not have HDF5 read the data directly into the application buffer by manipulating the selection.
    Instead, we read all data into a temporary buffer and copy them into the application buffer using MPI_Pack.
  + Bypassing HDF5 with MPI-IO
    An alternate solution to read data is to use MPI-IO directly.
    HDF5 selection can be easily translated to MPI derived datatype using subarray type (`MPI_Type_create_subarray`).
    We can adjust the order in memory datatype to read directly into the application buffer, assuming the application does not read the same place twice.
---

### Other Lessons Learned
#### Metadata operation cost
* Expensive B-tree operations --
  Everything in HDF5 is indexed using B-tree.
  There is a B-tree under each group to track data objects within that group.
  There is a B-tree for every chunked dataset to track its chunks.
  Locating data objects requires traveling the B-tree form the current location (e.g., the loc_id passed to H5Dopn).
  Since the location of a child node of a B-tree is not known before reading its parent node, traversing a B-tree may involve multiple writes.
  + We should avoid creating many hidden data object (datasets, groups)
    Metadata operations are relatively expensive compared to NetCDF.
    They may involve multiple read operations.
