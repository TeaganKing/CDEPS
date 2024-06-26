.. _input-streams:

Input Streams
=============

--------
Overview
--------

An *input data stream* is a time-series of input data files where all
the fields in the stream are located in the same data file and all
share the same spatial and temporal coordinates (ie. are all on the
same grid and share the same time axis). Normally a time axis has a
uniform dt, but this is not a requirement.

The data models can have multiple input streams.

The data for one stream may be all in one file or may be spread over
several files. For example, 50 years of monthly average data might be
contained all in one data file or it might be spread over 50 files,
each containing one year of data.

The data models can *loop* over stream data -- i.e., repeatedly cycle
over some subset of an input stream's time axis. When looping, the
models can only loop over whole years. For example, an input stream
might have SST data for years 1950 through 2000, but a model could
loop over the data for years 1960 through 1980. A model *cannot* loop
over partial years, for example, from 1950-Feb-10 through 1980-Mar-15.

The input data must be in a netcdf file and the time axis in that file
must be CF-1.0 compliant.

There are two main categories of information that the data models need
to know about a stream:

- data that describes what a user wants -- what streams to use and how
  to use them -- things that can be changed by a user.

- data that describes the stream data -- meta-data about the inherent
  properties of the data itself -- things that cannot be changed by a
  user.

Generally, information about what streams a user wants to use and how
to use them is input via the strdata ("stream data") Fortran namelist,
while meta-data that describes the stream data itself is found in an
xml-like text or ESMF config file called a "stream description file."

.. _stream_description_file:

-----------------------
Data Model Stream Input
-----------------------

The data models advance in time discretely.  At a given time, the
stream code is called to advance the data model by readind in fields
from input files.  Those input files have data on a discrete time axis
as well.  Each data point in the input files is associated with a
discrete time (as opposed to a time interval).  Depending on whether
you pick lower, upper, nearest, linear, or coszen, the data in the
input file will be "interpolated" to the time in the model.

XML format
----------

In case of using XML format, stream-dependent input is named as 
``d{model_name}.streams.xml``, where ``model_name`` can be ``atm``,
``ice``, ``lnd``, ``ocn``, ``rof`` or ``wav``.  Multiple streams can
be specified in the this XML file (see :ref:`streams<input-streams>`).

The schema for this xml file is as follows::

  <file id="stream" version="2.0">
    <stream_info>
      <taxmode></taxmode>
      <tintalgo></tintalgo>
      <readmode></readmode>
      <mapalgo></mapalgo>
      <dtlimit></dtlimit>
      <yearFirst></yearFirst>
      <yearLast></yearLast>
      <yearAlign></yearAlign>
      <vectors></vectors>
      <meshfile></meshfile>
      <lev_dimname></lev_dimname>
      <datafiles>
        <file></file>
      </datafiles>
      <datavars>
        <var></var>
      </datavars>
      <offset></offset>
    </stream_info>
  </file>

ESMF Config format
------------------

In case of using ESMF Config format, stream-dependent input is named as
``d{model_name}.streams`, where ``model_name`` can be ``atm``,
``ice``, ``lnd``, ``ocn``, ``rof`` or ``wav``. Again, multiple streams can
be specified in the this XML file (see :ref:`streams<input-streams>`).

The structure of ESMF config file is as follows::

  file_id: "stream"
  file_version: 2.0

  stream_info:

  taxmode{id}:
  tInterpAlgo{id}:
  readMode{id}:
  mapalgo{id}:
  dtlimit{id}:
  yearFirst{id}:
  yearLast{id}:
  yearAlign{id}:
  stream_vectors{id}:
  stream_mesh_file{id}:
  stream_lev_dimname{id}:
  stream_data_files{id}:
  stream_data_variables{id}:
  stream_offset{id}:

In this case `{id}` is a formatted number like `01`, `02`, etc. that is
used distinguish among each streams in the same data mode. The `stream_info`
part will include multiple entry in case of using multiple stream to define
data model. For example, `stream_info` can be structured as `stream_info: 
CLMGSWP3v1.Solar01 CLMGSWP3v1.Precip02 CLMGSWP3v1.TPQW03 topo.observed04` for
`CLMNCEP` data mode and other namelist options can be repeated for all streams
by using `01`, `02`, `03`, `04` as `id` in each stream respectively.

Definitions of each keys used in stream definition file
-------------------------------------------------------

**taxmode**
  How to handle data outside the specified stream time axis.
  Valid options are to cycle the data based on the first, last, and align
  settings associated with the stream dataset, to extend the first and last
  valid value indefinitely, or to limit the interpolated data to fall only between
  the least and greatest valid value of the time array. Valid values are:

  *extend* = extrapolate before and after the period by using the first or last value.

  *cycle*  = cycle between the range of data

  *limit*  = restrict to the period for which the data is valid

.. note::
  CIME-CCS default is *cycle*

**tintalgo**
  Specifies time interpolation algorithm option. Valid values are:

  *lower*   = Use lower time-value

  *upper*   = Use upper time-value

  *nearest* = Use the nearest time-value

  *linear*  = Linearly interpolate between the two time-values

  *coszen*  = Scale according to the cosine of the solar zenith angle (for solar)

.. note::
  CIME-CCS default is *linear*

**readmode**
  Specifies data stream read mode. The valid values are:

  *single*    = Reads single record in each time

  *full_file* = Read entire file (might be memory consuming for high-resolution cases)

.. note::
  CIME-CCS default is *single*

**mapalgo**
  Specifies spatial interpolation algorithm to map stream data on stream mesh
  to stream data on model mesh. The used interpolation algorithm is based on the ones
  that are provided by ESMF library. Valid values are:

  *redist*   = Redistributes data from source mesh to destination mesh

  *nn*       = In this version of nearest neighbor interpolation each destination point is 
  mapped to the closest source point. A given source point may go to multiple 
  destination points, but no destination point will receive input from more 
  than one source point.

  *bilinear* = Bilinear interpolation. Destination value is a linear combination of the 
  source values in the cell which contains the destination point. The weights 
  for the linear combination are based on the distance of destination point 
  from each source value.

  *consd*    = First-order conservative interpolation. The main purpose of this method is 
  to preserve the integral of the field between the source and destination. Tt uses 
  destination area normalization (*ESMF_NORMTYPE_DSTAREA*). Here the weights are 
  calculated by dividing the area of overlap of the source and destination cells by the 
  area of the entire destination cell.

  *consf*    = Same with *consd* but in this case it uses fraction area normalization 
  (*ESMF_NORMTYPE_FRACAREA*). Here in addition to the weight calculation done for 
  destination area normalization the weights are also divided by the fraction that the 
  destination cell overlaps with the entire source grid.

.. note::
  CIME-CCS default is *bilinear*

.. note::
  More information about ESMF provided regridding options can be found in `here 
  <http://earthsystemmodeling.org/docs/nightly/develop/ESMF_refdoc/node5.html#sec:regrid>`_.

**dtlimit**
  Specifies delta time ratio limits placed on the time interpolation
  associated with the array of streams. Causes the model to stop if
  the ratio of the running maximum delta time divided by the minimum delta time
  is greater than the dtlimit for that stream.

  For instance, with daily data, the delta time should be exactly one
  day throughout the dataset and the computed maximum divided by
  minimum delta time should always be 1.0. For monthly data, the
  delta time should be between 28 and 31 days and the maximum ratio
  should be about 1.1. The running value of the delta time is
  computed as data is read and any wraparound or cycling is also
  included. This input helps trap missing data or errors in cycling.
  to turn off trapping, set the value to 1.0e30 or something similar.
  In case of very high-resolution temporal datasets (i.e. hourly), 
  the limit might need to be set to a value greater than 1.0 due to the
  round-off arithmetic.

.. note::
  CIME-CCS default is set to 1.5

**yearFirst**
  The first year of stream data that will be used

**yearLast**
  The last year of stream data that will be used

**yearAlign**
  The simulation year corresponding to ``yearFirst``.

  A common usage is to set this to the first year of the model run
  (for CIME-CCS this would correspond to the xml variable
  ``RUN_STARTDATE``). With this setting, the forcing in the first year
  of the run will be the forcing of year ``yearFirst``.

  Another usage is to align the calendar of transient forcing with
  the model calendar. For example, setting ``yearAlign`` =
  ``yearFirst`` will lead to the forcing calendar being the same as
  the model calendar. The forcing for a given model year would be the
  forcing of the same year. This would be appropriate in transient
  runs where the model calendar is setup to span the same year range
  as the forcing data.

.. note::
  The following pertains to CIME-CCS details for yearAlign usage.

  For some data model modes, ``yearAlign`` can be set via an xml variable
  whose name ends with ``YR_ALIGN`` (there are a few such xml variables,
  each pertaining to a particular data model mode).

  An example of this is land-only historical simulations in which we run
  the model for 1850 to 2010 using atmospheric forcing data that is only
  available for 1901 to 2010. In this case, we want to run the model for
  years 1850 (so ``RUN_STARTDATE`` has year 1850) through 1900 by looping
  over the forcing data for 1901-1920, and then run the model for years
  1901-2010 using the forcing data from 1901-2010. To do this, initially set::

    ./xmlchange DATM_YR_ALIGN=1901
    ./xmlchange DATM_YR_START=1901
    ./xmlchange DATM_YR_END=1920

  When the model has completed year 1900, set::

    ./xmlchange DATM_YR_ALIGN=1901
    ./xmlchange DATM_YR_START=1901
    ./xmlchange DATM_YR_END=2010

  With this setup, the correlation between model run year and forcing year
  looks like this::

    RUN   Year : 1850 ... 1860 1861 ... 1870 ... 1880 1881 ... 1890 ... 1900 1901 ... 2010
    FORCE Year : 1910 ... 1920 1901 ... 1910 ... 1920 1901 ... 1910 ... 1920 1901 ... 2010

  Setting ``DATM_YR_ALIGN`` to 1901 tells the code that you want
  to align model year 1901 with forcing data year 1901, and then it
  calculates what the forcing year should be if the model starts in year 1850.

**vectors**
  Specifies paired vector field names that will be rotated to make them 
  relative to earth coordinates using source mesh coordinates. Then,
  rotated fields are remapped to the destination mesh. Rotating fields in the
  destination mesh to make them relative to the destination model mesh 
  is the destination component responsibilty. For example, the valid value
  for *vectors* argument could be *"Sa_u:Sa_v"*. In the current version,
  there is no way to specify multiple vecors fields in same data stream.

.. note::
  CIME-CCS default is set to null

**meshfile**

  Specifies filename for mesh for all fields on the stream.

**lev_dimname**

  Specifies name of vertical dimension in stream.

.. note::
  CIME-CCS default is set to null

**datafiles**
  In case of usage of XML format to define data stream, each 
  <file></file> entry contains a data files to use. If there is
  more than one file, the files must be in chronological order, that
  is, the dates in time axis of the first file are before the dates
  in the time axis of the second file.

  The same rule also applies to the ESMF config format but in this
  case the files needs to be listed by seperating them with spaces.

**datavars**
  In case of usage of XML format to define data stream, each
  <var></var> entry contains a paired list with the name of the
  variable in the netCDF file on the left and the name of the
  corresponding model variable on the right.

  The same rule also applies to the ESMF config format but in this 
  case the variable pairs needs to be listed by seperating
  them with spaces.

**offset**
  The offset allows a user to shift the time axis of a data stream by
  a fixed and constant number of seconds. For instance, if a data set
  contains daily average data with timestamps for the data at the end
  of the day, it might be appropriate to shift the time axis by 12
  hours so the data is taken to be at the middle of the day instead of
  the end of the day. This feature supports only simple shifts in
  seconds as a way of correcting input data time axes without having
  to modify the input data time axis manually. This feature does not
  support more complex shifts such as end of month to mid-month. But
  in conjunction with the time interpolation methods in the strdata
  input, hopefully most user needs can be accommodated with the two
  settings. Note that a positive offset advances the input data time
  axis forward by that number of seconds.

  As an example of offsets, if the input data is at 0, 3600, 7200,
  10800 seconds (hourly) and you set an offset of 1800, then the input
  data will be set at times 1800, 5400, 9000, and 12600. So a model
  at time 3600 using linear interpolation would have data at "n=2"
  with offset of 0 will have data at "n=(2+3)/2" with an offset
  of 1800. n=2 is the 2nd data in the time list 0, 3600, 7200, 10800
  in this example. n=(2+3)/2 is the average of the 2nd and 3rd data
  in the time list 0, 3600, 7200, 10800. Offset can be positive or
  negative.

.. note::
  CIME-CCS default is set to 0

.. _input-namelists:

-------------------------
Data Model Namelist Input
-------------------------

Each data model has an associated input namelist file, ``d{model_name}_in``,
where ``model_name=[datm,dlnd,dglc,dice,docn,drof,dwav]``.

The following namelist variables appear in each data model namelist:

**dataMode**
  Component specific mode.

.. note::
  Each data model has its own datamode values as described below:

  :ref:`Data Atmosphere <datm>`

  :ref:`Data Ice <dice>`

  :ref:`Data Land <dlnd>`

  :ref:`Data Land-Ice <dglc>`

  :ref:`Data Ocean <docn>`

  :ref:`Data Runoff <drof>`

  :ref:`Data Wave <dwav>`

---------------------------------------------------
 CIME-CCS Customization of stream description files
---------------------------------------------------

Each data model's **cime-config/buildnml** utility automatically
generates the required stream description files for the case.  The
directory contents of each data model will look like the following,
where ``model_name`` can be ``atm``, ``ice``, ``lnd``, ``ocn``,
``rof`` or ``wav`` ::

   $CIMEROOT/components/cdeps/{model_name}/cime_config/buildnml
   $CIMEROOT/components/cdeps/{model_name}/cime_config/namelist_definition_{model_name}.xml
   $CIMEROOT/components/cdeps/{model_name}/cime_config/stream_definition_{model_name}.xml

The ``namelist_definition_{model_name}.xml`` file defines and sets default
values for all the namelist variables and associated groups and also
provides out-of-the box settings for the target data model and target
stream.  **buildnml** utilizes these two files to construct the stream
files for the given compset settings. You can modify the generated
stream files for your particular needs by doing the following:

1. Copy the relevant description file from ``$CASEROOT/CaseDocs`` to
   ``$CASEROOT``. Change the permission of the file to write.

2. Edit the ``$CASEROOT`` file with your desired changes.

   - *Be sure not to put any tab characters in the file: use spaces
     instead*.

3. Call **preview_namelists** and verify that your changes do indeed
   appear in the resultant stream description file appear in
   ``CaseDocs/{model_name}streams.xml``. These changes will
   also appear in ``$RUNDIR/{model_name}.streams.xml``.

The ``stream_definition_{model_name}.xml`` file defines and sets default
values for stream data sources such as list of variable pairs, stream data files
spatial and temporal interpolation types. The file can be modified by
folloiwng same approach that is used to modify
``namelist_definition_{model_name}.xml`` file.

----------------------------
Data Model Stream Inline API
----------------------------

As mentioned previously, the streams code can be used from either a
CDEPS data model **OR** inline calls from a prognostic component. This
is a very powerful feature in that data model input can be obtained
using standardized interfaces and ESMF online mapping.

The inline API assumes that there is **only one stream** and consists
of two calls: one to initialize the stream data type
(``shr_strdata_init``):

.. code-block:: Fortran

    subroutine shr_strdata_init_from_inline(sdat, my_task, logunit, compname, &
       model_clock, model_mesh, stream_meshfile, stream_lev_dimname, stream_mapalgo, &
       stream_filenames, stream_fldlistFile, stream_fldListModel, &
       stream_yearFirst, stream_yearLast, stream_yearAlign, &
       stream_offset, stream_taxmode, stream_dtlimit, stream_tintalgo, stream_name, rc)

    ! input/output variables
    type(shr_strdata_type) , intent(inout) :: sdat                   ! stream data type
    integer                , intent(in)    :: my_task                ! my mpi task
    integer                , intent(in)    :: logunit                ! stdout logunit
    character(len=*)       , intent(in)    :: compname               ! component name (e.g. ATM, OCN, ...)
    type(ESMF_Clock)       , intent(in)    :: model_clock            ! model clock
    type(ESMF_Mesh)        , intent(in)    :: model_mesh             ! model mesh
    character(*)           , intent(in)    :: stream_meshFile        ! full pathname to stream mesh file
    character(*)           , intent(in)    :: stream_lev_dimname     ! name of vertical dimension in stream
    character(*)           , intent(in)    :: stream_mapalgo         ! stream mesh -> model mesh mapping type
    character(*)           , intent(in)    :: stream_filenames(:)    ! stream data filenames (full pathnamesa)
    character(*)           , intent(in)    :: stream_fldListFile(:)  ! file field names, colon delim list
    character(*)           , intent(in)    :: stream_fldListModel(:) ! model field names, colon delim list
    integer                , intent(in)    :: stream_yearFirst       ! first year to use
    integer                , intent(in)    :: stream_yearLast        ! last  year to use
    integer                , intent(in)    :: stream_yearAlign       ! align yearFirst with this model year
    integer                , intent(in)    :: stream_offset          ! offset in seconds of stream data
    character(*)           , intent(in)    :: stream_taxMode         ! time axis mode
    real(r8)               , intent(in)    :: stream_dtlimit         ! ratio of max/min stream delta times
    character(*)           , intent(in)    :: stream_tintalgo        ! time interpolation algorithm
    character(*), optional , intent(in)    :: stream_name            ! name of stream
    integer                , intent(out)   :: rc                     ! error code

and one to advance the stream (``shr_strdata_advance``):

.. code-block:: Fortran

    subroutine shr_strdata_advance(sdat, ymd, tod, logunit, istr, timers, rc)

    type(shr_strdata_type) ,intent(inout)       :: sdat    ! stream data type
    integer                ,intent(in)          :: ymd     ! current model date
    integer                ,intent(in)          :: tod     ! current model date
    integer                ,intent(in)          :: logunit ! stdout logunit
    character(len=*)       ,intent(in)          :: istr    ! string used for timing output
    logical                ,intent(in), optional:: timers  ! currently not used
    integer                ,intent(out)         :: rc      ! error code

-------------------------
Handling Stream Calendars
-------------------------

Handling stream calendars can be tricky if there are mismatches
between the stream and data model calendars. CDEPS always uses the
stream calendar for time interpolation for reasons described below.
When there is a calendar mismatch, Feb 29 is supported in a special
way as needed to get reasonable values.  Note that when Feb 29 needs
to be treated specially, a discontinuity will be introduced. The size
of that discontinuity will depend on the time series input data.
Four cases can occur:

1. The stream calendar and model calendar are identical and CDEPS
   proceeds in the standard way.

2. The stream has a no leap calendar and the model is on the gregorian calendar.
   Time interpolation is performed on the noleap calendar. If the model date is Feb 29,
   the code computes stream data for Feb 28 by setting model time to Feb 28.
   This results in duplicate stream data on Feb 28 and Feb 29 and a
   discontinuity at the start of Feb 29. This could potentially be fixed
   by using the gregorian calendar for time interpolation when the input data
   is relatively infrequent (say greater than daily) with the following concerns.

   - The forcing will not be reproduced identically on the same day with
     with climatological inputs data

   - Input data with variable input frequency might behave funny

   - An arbitrary discontinuity will be introduced in the time
     interpolation method based upon the logic chosen to transition
     from reproducing Feb 28 on Feb 29 and interpolating to Feb 29.

   - The time gradient of data will change by adding a day arbitrarily.

3. The stream is a gregorian calendar and the model is a noleap calendar
   Time interpolate on the gregorian calendar. This causes Feb 29
   stream data to be skipped and lead to a discontinuity at the start
   of March 1.

4. The calendars mismatch and none of the above
   If the calendars mismatch and neither of the three cases above are
   recognized, then abort.
