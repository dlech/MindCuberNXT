MindCuber
=========

By David Gilday

Copyright (C) 2011-2013 David Gilday

http://mindcuber.com

Release Information: v2.2

Release History:  
 - v2.2 - 13th July 2013, include support for cubes with black stickers  
 - v2.1 - 22nd June 2013, initial public release

Usage:
-----

  This software may be used for any non-commercial purpose providing
  that the original author is acknowledged.

Disclaimer:
----------

  This software is provided 'as is' without warranty of any kind, either
  express or implied, including, but not limited to, the implied warranties
  of fitness for a purpose, or the warranty of non-infringement.

Description:
-----------

MindCuber is a robot that can be built from LEGO MINDSTORMS NXT to solve a
Rubik's Cube.

This release includes source code for MindCuber that can be compiled for both
the NXT 2.0 variant and for the "Orange Box" #8527 variant. The source may also
be configured to allow MindCuber to "solve" the cube to one or more patterns
rather than the normal solved state.

Using BricxCC:
-------------

1. Install BricxCC on your computer (see notes below for link)

2. Open BricxCC

3. Open the menu Edit/Preferences..., select the NBC/NXC tab and
   set the preferences recommended in the notes below

4. Use menu Tools/Find Brick... to connect to the NXT

5. Use the menu File/Open... to open the MindCuber.nxc source file

6. If you wish to compile for the #8257 variant of MindCuber, remove the
   comment characters from the line in MindCuber.rxe that says:

        //#define NXT_8527
    
    to say:

        #define NXT_8527

7. Select menu Compile/Download and Run to compile the program, download
   it to the NXT and run

The source includes some other options for creating patterns by uncommenting
other lines near the top of the MindCuber.nxc.

The source includes options to solve cubes with black stickers rather than
white. Uncomment one or both of the following lines as appropriate:

    //#define BLACK_FACE
    //#define WHITE_LOGO

The option for using the HiTechnic v2 color sensor as an alternative to the
LEGO color sensor has not been tested so may need some additional work.

Please see http://mindcuber.com for further information.

Notes:
-----

MindCuber.nxc can be compiled with Bricx Command Center (BricxCC) available 
from: http://sourceforge.net/projects/bricxcc/files/bricxcc and has been
tested with BricxCC version 3.3 (Build 3.3.8.9)

The compiled program, MindCuber.rxe, will run on standard LEGO MINDSTORMS NXT
firmware and has been tested with version 1.31

The MindCuber program uses almost all of the available memory in the NXT so
any changes to the code which increase its' size may cause issues. The choice
of firmware on the NXT may affect how much memory is available so other versions
of the NXT firmware may also cause issues.

BricxCC preferences for compiling with NBC/NXC:
  + nbc-1.2.1r4
  + NXT 2.0 compatible firmware
  + Automatic firmware version
  + Optimization level 2

Not tested with:
  - internal compiler
  - enahanced firmware
