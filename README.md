# CM1 Tendency Nudging Initiation Technique

This folder contains information on the updraft tendency nudging (TN) initiation technique for Cloud Model 1 (CM1) as developed and described in further detail in ([Flournoy and Rasmussen (2023; MWR)](https://journals.ametsoc.org/view/journals/mwre/151/9/MWR-D-22-0069.1.xml)). Documented changes to the technique can be found in `VERSIONS.txt`, although the implementation of this technique can be done entirely by the user.

## Table of Contents
- [Repository Structure](#repository-structure)
- [About Tendency Nudging](#about-tendency-nudging)
- [Example Build](#example-build)
- [BYO CM1](#byo-cm1)
- [Other Notes](#other-notes)

## Repository Structure

For users new to CM1, the `tendency_nudging` folder contains several files and folders that can be used to implement the TN into CM1. Below is an overview of the main repository structure:

- **`/FR_MWR/*`**: Directory containing an example namelist (`/FR_MWR/namelist-018.input`), tendency nudging input file (`/FR_MWR/toy.input-018.input`), and input sounding (`/FR_MWR/input_sounding-018`) for the 0.018 K/s simulation in Flournoy and Rassmusen (2023). Differences in the file parameters between simulations can be found ([here](https://journals.ametsoc.org/view/journals/mwre/151/9/MWR-D-22-0069.1.xml)).
- **`/cm1r21.1_tn.zip`**: Compressed directory containing a ready-to-go build of CM1 (v21.1). Please see [the Example Build section](#example-build) for more details on using this build.
- **`toy.input_readme`**: Readme file specifying the variables found in `toy.input`.
- **`toy.input`**: File used to drive the initiation technique containing all user-defined variables for implementing the technique in simulations.
- **`VERSIONS.txt`**: A record of changes made to this repository. Check this file to see updates or modifications.


## About Tendency Nudging 
While more details can be found in ([FR23](https://journals.ametsoc.org/view/journals/mwre/151/9/MWR-D-22-0069.1.xml)), using the TN initiation mechanism is designed to provide a more 'gentle' and 'realistic' initiation process. This technique was designed to also prevent undesired (and unrealistic) characteristics like early tornado-like vortices (TLVs).

The variables that drive the TN technique can be defined in `toy.input`. They can be found in `toy.input_readme`, and as follows: 

- **warmamp**: max amplitude of heating (K/s)
- **warmxc**: x-coordinate representing the center of the theta tendency (m)
- **warmyc**: same as above for y-coordinate 
- **warmzc**: same as above for z-coordinate
- **warmrad**: horizontal radius of the bubble (m)
- **warmdepth**: vertical radius of the bubble (m)
- **warmtime**: model time to begin heating (s)
- **warmtime2**: model time at which max amplitude is reached (s)
- **warmtime3**: model time to begin cooling to 0 tendency (s)
- **warmtime4**: model time where the heating tendency has returned to 0 K/s (s)

**NOTE**: This pattern is maintained for two *additional* warm bubbles and one cold bubble in the provided `toy.input`. The only difference between the warm bubble and cold bubble application is to ensure that $coldamp < 0$. 

## Example Build 

Users can use the provided cm1 build to run simulations **_without_** needing to edit the source code. Currently, this is limited to version 21.1, although other versions may be added in the future. Please follow the provided steps (once the contents have been unzipped) to compile and prepare for a simulation:

1. **copy CM1 to your device**: First, we'll need to ensure that the cm1 code is on the device that you plan to run the simulations on. Whether you're on a supercomputer, or locally, the easiest way to achieve this will be to clone this repository to you device by doing the following:
``` bash 
cd /path/to/dir/
git clone <link_to_repo_here>
```

2. **import modules/packages**: These packages are what allow CM1 to run. Packages like netCDF will be needed whether in a python environment or via supercomputer modules will be needed to run the model. There is not once answer that fits all machines, so every computing method will have its own 'secret recipe'.

3. **compile the source code**: To run simulations, you'll need to compile the source code of the model by first editing `Makefile` to match the requirements of the computer you'll be running the simulations on. Note the current format of the Makefile is designed for NCAR's Derecho supercomputer. Once `Makefile` has been edited accordingly, please compile the code. You can do so by running:

``` bash
cd cm1r21.1_tn/src/ # navigate to the directory containing the Makefile
vi Makefile # open the Makefile for editing
make # compiles the Fortran code
```  

4. **edit `namelist.input`**: You'll now need to prescribe your desired settings for the model. When editing, make sure that `iinit = 0` to ensure stock initiation techniques attempt to run concurrently with the tendency nudging (this is because `toy.input` and the source code changes will be driving the initiation). If navigating to `namelist.input` directly after compiling code, you can do so by running:

```bash
cd ../run # change to dir containing namelist.input
vi namelist.input # open file for editing
```

5. **edit `toy.input`**: This is where you can customize the parameters for the tendency nudging initiation mechanism. See [mechanism details](#mechanism_details) for more information on the variables provided and [other notes](#other-notes) for more things to be cognizant of when using this technique. This file is in the same `/run` dir as `namelist.input`, so just use `vi` or `nano` to edit this file; no additional commands are needed.

6. **see additional resources (if needed)**: For additional notes on running and compiling CM1, please visit the ([models webpage](https://www2.mmm.ucar.edu/people/bryan/cm1/user_guide_brief.html)).

You are now set to run a simulation using the tendency nudging technique. Use the `cm1.exe` file in the `/run` dir to execute the model. As a reminder, `toy.input` must be in the same directory as the other necessary files to execute the model (`LANDUSE.TBL`, `namelist.input`, `input_sounding`, `cm1.exe`), but these do not need to permanently reside in the `/run` dir.

**NOTE**: This provided build supports up to three warm bubbles and one cold bubble.

## BYO CM1
Although TN implemented into v21.1 is provided, TN can and has been added to various historical versions of the model (e.g., 19.5). Adding the capability to utilize TN in the model requires code to be added to a variety of source code files. Before editing the source code, it is recommended to have an *unedited* version of the model to roll back to if needed. The code required to add this capability is as follows:

- `/src/cm1.F/`: This file requires additions in two places. First, we'll define the variables and read in the `toy.input` parameters after configuring MPI details. Second, we'll provide solve1.F the variables needed to apply the technique. The lines in which these additions are made are approximately 277 and 3319, although these will vary based on the version used. The chunks of code are found below:

``` fortran
! Toy model variables

      real :: bigZ, bigR, warmrad, warmdepth                                    ! TN
      real :: warmtime2, warmtime3, warmtime4                                   ! TN
      real :: coldtime2, coldtime3, coldtime4                                   ! TN
      real :: warm0amp,warm0xc,warm0yc,warm0zc,warm0rad,warm0depth              ! TN
      real :: warm0time,warm0time2,warm0time3,warm0time4                        ! TN
      real :: warm1amp,warm1xc,warm1yc,warm1zc,warm1rad,warm1depth              ! TN
      real :: warm1time,warm1time2,warm1time3,warm1time4                        ! TN
      real :: warmxc, warmyc, warmzc, warmamp, radius                           ! TN
      real :: coldrad, colddepth, coldxc, coldyc, coldzc, coldamp               ! TN
      real :: coldtime, warmtime                                                ! TN
      namelist /toyparms/ warmamp,warmxc,warmyc,warmzc,warmrad,warmdepth, &     ! TN
               warmtime,warmtime2,warmtime3,warmtime4, &                        ! TN
               warm0amp,warm0xc,warm0yc,warm0zc,warm0rad,warm0depth, &          ! TN
               warm0time,warm0time2,warm0time3,warm0time4, &                    ! TN
               coldamp,coldxc,coldyc,coldzc,coldrad,colddepth, &                ! TN
               coldtime,coldtime2,coldtime3,coldtime4                           ! TN

!----------------------------------------------------------------------
! Read-in toy model parameters

      open(unit=99,file='toy.input',form='formatted',status='old',    &         ! TN
           access='sequential')                                                 ! TN
      read(99,nml=toyparms)                                                     ! TN
      close(unit=99)                                                            ! TN
```

``` fortran
    warmamp,warmxc,warmyc,warmzc,warmrad,warmdepth,warmtime,warmtime2,warmtime3,warmtime4, &               ! TN
    warm0amp,warm0xc,warm0yc,warm0zc,warm0rad,warm0depth,warm0time,warm0time2,warm0time3,warm0time4, &     ! TN
    warm1amp,warm1xc,warm1yc,warm1zc,warm1rad,warm1depth,warm1time,warm1time2,warm1time3,warm1time4, &     ! TN
    coldamp,coldxc,coldyc,coldzc,coldrad,colddepth,coldtime,coldtime2,coldtime3,coldtime4                  ! TN
```

- `/src/solve1.F/`: The additions made to this file will apply the initiation mechanism to the model variables. There are several additions that need to be made to this file. First, we must read in the variables being provided to the file (from `cm1.F`). Second, we will be defining several variables that are created via `toy.input` to be used in the model. Third, we will be applying the appropriate heating to the theta tendency. **NOTE**: the third section of code will need to be copied and pasted subsequently for additional warm bubbles (`toy.input` and the source code changes support three warm bubbles with the same number of parameters (e.g., warmxc, warm0xc, warm1xc)), so if you wanted to utilize all three, then the third section would need to be copied and pasted two *additional* times and variable names would need to be changed accordingly. The fourth section is similar to the third although it implements the cold bubble. The lines in which these additions are made are approximately 115, 236, 1159, and the cold bubble chunk should directly follow all additions made surrounding the warm bubbles. Again, these will vary based on the version used.

``` fortran
    warmamp,warmxc,warmyc,warmzc,warmrad,warmdepth,warmtime,warmtime2,warmtime3,warmtime4,           &   ! TN
    warm0amp,warm0xc,warm0yc,warm0zc,warm0rad,warm0depth,warm0time,warm0time2,warm0time3,warm0time4, &   ! TN
    warm1amp,warm1xc,warm1yc,warm1zc,warm1rad,warm1depth,warm1time,warm1time2,warm1time3,warm1time4, &   ! TN
    coldamp,coldxc,coldyc,coldzc,coldrad,colddepth,coldtime,coldtime2,coldtime3,coldtime4                ! TN
```

``` fortran
! Toy model variables

      real :: bigZ, bigR, warmrad, warmdepth                        ! TN
      real :: warmxc, warmyc, warmzc, warmamp, radius               ! TN
      real :: coldrad, colddepth, coldxc, coldyc, coldzc, coldamp   ! TN
      real :: warmtime, coldtime                                    ! TN
      real :: warmtime2, warmtime3, warmtime4                       ! TN
      real :: coldtime2, coldtime3, coldtime4                       ! TN
      real :: warmmod, coldmod                                      ! TN
      real :: warm0amp,warm0xc,warm0yc,warm0zc                      ! TN
      real :: warm0rad,warm0depth,warm0mod                          ! TN
      real :: warm0time,warm0time2,warm0time3,warm0time4            ! TN
      real :: warm1amp,warm1xc,warm1yc,warm1zc                      ! TN
      real :: warm1rad,warm1depth,warm1mod                          ! TN
      real :: warm1time,warm1time2,warm1time3,warm1time4            ! TN
      real :: newwarmxc,newwarmyc                                   ! TN
      real :: newwarm0xc,newwarm0yc                                 ! TN
      real :: newwarm1xc,newwarm1yc                                 ! TN       
      real :: newcoldxc,newcoldyc                                   ! TN
```

``` fortran 
      ! Toy model heat source/sink stuff...                                     ! TN                    

      ! heat source                                                             ! TN      
      ! on @ warmtime, ramps up until warmtime2                                 ! TN      
      ! max amp until warmtime3, ramps down until warmtime4                     ! TN      

      ! Adjust for domain translation to keep heat source position constant     ! TN      
      ! (translate with the domain)                                             ! TN      
      newwarmxc = warmxc - (mtime*umove)                                        ! TN      
      newwarmyc = warmyc - (mtime*vmove)                                        ! TN      

      if (mtime.ge.warmtime.and.mtime.lt.warmtime4) then                        ! TN      
        if(mtime.lt.warmtime2) then                                             ! TN      
          warmmod = (mtime - warmtime) / (warmtime2 - warmtime)                 ! TN      
        elseif(mtime .ge. warmtime2 .and. mtime.lt.warmtime3) then              ! TN      
          warmmod = 1.0                                                         ! TN      
        elseif(mtime .ge. warmtime3 .and. mtime.lt.warmtime4) then              ! TN      
          warmmod = (warmtime4 - mtime) / (warmtime4 - warmtime3)               ! TN      
        else                                                                    ! TN      
          warmmod = 0.0                                                         ! TN      
        endif                                                                   ! TN      

        do k=1,nk                                                               ! TN
        do j=1,nj                                                               ! TN
        do i=1,ni                                                               ! TN
          radius = ((xh(i)-newwarmxc)**2. + (yh(j)-newwarmyc)**2.)**0.5         ! TN
          if (radius .lt. warmrad) then                                         ! TN 
            BigR = 1 - (radius**2./warmrad**2.)                                 ! TN
          else                                                                  ! TN
            BigR = 0.0                                                          ! TN
          endif                                                                 ! TN
          radius = abs(zh(i,j,k)-warmzc)                                        ! TN
          if (radius .lt. warmdepth) then                                       ! TN
            BigZ = 1 - (radius**2./warmdepth**2.)                               ! TN
          else                                                                  ! TN
            BigZ = 0.0                                                          ! TN
          endif                                                                 ! TN
          thten1(i,j,k) = thten1(i,j,k) + warmamp * bigR * bigZ * warmmod       ! TN
        enddo                                                                   ! TN
        enddo                                                                   ! TN
        enddo                                                                   ! TN
      endif                                                                     ! TN
```

``` fortran 
! heat sink (turned on at coldtime)

      if (mtime.ge.coldtime.and.mtime.lt.coldtime4) then

      if(mtime.lt.coldtime2) then
         coldmod = (mtime- coldtime) / (coldtime2 - coldtime)
      elseif(mtime.lt.coldtime3) then
         coldmod = 1.0
      elseif(mtime.lt.coldtime4) then
         coldmod = (coldtime4 - mtime) / (coldtime4 - coldtime3)
      else
         coldmod = 0.0
      endif

      do k=1,nk
      do j=1,nj
      do i=1,ni
         radius = ((xh(i)-coldxc)**2. + (yh(j)-coldyc)**2.)**0.5
         if (radius .lt. coldrad) then
            BigR = 1 - (radius**2./coldrad**2.)
         else
            BigR = 0.0
         endif
         radius = abs(zh(i,j,k)-coldzc)
         if (radius .lt. colddepth) then
            BigZ = 1 - (radius**2./colddepth**2.)
         else
            BigZ = 0.0
         endif

         thten1(i,j,k) = thten1(i,j,k) +    &
                    coldamp * bigR * bigZ * coldmod

      enddo
      enddo
      enddo

      endif
```

## Other Notes
- **# of bubbles**: As mentioned previously, the current version of `toy.input` allows for up to 3 warm bubbles and 1 cold bubble. However, a user can define virtually any number of warm or cold bubbles first by defining the varaibles and their values in `toy.input`. Then, the user would need to follow the steps here and make additions to the source code as needed to fit the number of bubble perscribed in `toy.input`. 

- **cooking boundary layers (if u/v move != 0)**: As more users develop turbulence in CM1 via cooking a boundary layer, determining where to place the initial warm/cold bubble is critically important. Specificlally, the input x/y coordinates of the bubble must account for the time elapsed in addition to the domain translation. For determining the warmxc/yc positions to be entered into `toy.input` please use $$warmxc/warmyc = desired\:position\:(m) + [u/v_{move} * time\:of\:cook\:(s)]$$
This file supports the use of up to three warm and one cold bubble. Users can set the amplitude of non-used bubbles to 0.0 (e.g., warm0amp, coldamp) to prevent additional features from appearing the simulations.

Please contact ([Matt Flournoy](mailto:matthew.flournoy@noaa.gov)) if you have any questions, concerns, or beer offerings. 
