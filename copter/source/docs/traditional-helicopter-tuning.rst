.. _traditional-helicopter-tuning:

===============================
Traditional Helicopter – Tuning
===============================

This tuning guide provides initial preparation for tuning and instructions for manual tuning and autotune (4.2 and later).

For making setting changes to traditional helicopters, users are reminded to 
use only the Full Parameter List or Tree in your ground station software. 
**Do not use the Basic, Extended or Advanced Tuning pages that are designed for
multi-rotor aircraft.** These pages will make unwanted setting changes to 
traditional helicopters. And remember to write the changes to the flight 
controller after making them or they won't be saved!

Tuning Instructions
===================

.. toctree::
    :maxdepth: 1

    Manual Tuning Wiki <traditional-helicopter-manual-tuning>
    Autotune Wiki <traditional-helicopter-autotune>

General ArduCopter Flight Control Law Description
=================================================
Users should generally understand the flight control laws before tuning. At
a high level, the arducopter control laws are designed as a model following
architecture where the software converts the pilot input into a commanded
attitude (Stabilize Mode) or commanded rate (Acro mode) and controls the
aircraft to achieve that commanded value. In the background, the software keeps
track of, or predicts, where the aircraft should be in space (i.e. pitch and
roll attitude) based on the inputs of the pilot or autopilot. It has two
controllers (attitude and rate) that work together to ensure the actual aircraft
is following the software’s predicted pitch and roll rates and attitudes.
 
The pilot’s commands are limited by the amount of acceleration that can be
commanded through the :ref:`ATC_ACCEL_P_MAX<ATC_ACCEL_P_MAX>` for pitch and :ref:`ATC_ACCEL_R_MAX<ATC_ACCEL_R_MAX>` for roll.
The initial responsiveness (crispness/sluggishness) of the aircraft to the pilot
input can be adjusted through the :ref:`ATC_INPUT_TC<ATC_INPUT_TC>` parameter (in AC 3.5 or earlier,
this parameter was called RC_FEEL). The pilot input and these parameters are
used to determine the requested rate required to achieve the desired response
that is fed to the rate controller.
 
The attitude controller is used to ensure the actual attitude of the aircraft
matches the predicted attitude of the autopilot. It uses the
:ref:`ATC_ANG_PIT_P<ATC_ANG_PIT_P>` in pitch and the :ref:`ATC_ANG_RLL_P<ATC_ANG_RLL_P>` in roll to determine a rate that is
fed to the rate controller that will drive the aircraft to the predicted
attitude. 

The rate controller receives the sum of the requested rate resulting
from the pilot input and the rate from the attitude controller and determines
the swashplate commands required to achieve the input rate. The rate controller
uses a PID control algorithm and a feed forward path to control the aircraft and
achieve the input rate. The feed forward path uses the input rate and applies
the :ref:`ATC_RAT_PIT_VFF<ATC_RAT_PIT_VFF>` gain for pitch and :ref:`ATC_RAT_RLL_VFF<ATC_RAT_RLL_VFF>` gain for roll to
determine its portion of the swashplate command. The PID algorithm uses the
error between the actual rate and input rate to determine its portion of the
swashplate command. These are summed and sent to the mixing unit where the servo
positions are determined.

So this tuning method uses the FF gain initially to ensure the requested rates
match the actual rates.  However the rates can vary from the requested due to
disturbances. The P and D gains are then used to guard against disturbances
that cause the actual rates to deviate from the requested rates. So the P and D
gain may not be able to keep the actual rates exactly matching the requested
rates.  Since the software is tracking where the orientation of the aircraft
should be, then any error between the requested and actual rates will result in
attitude error. So there is a feature called the integrator that continually
sums the rate errors which effectively calculates the error in attitude.  The
I gain is multiplied by the integrator and summed with the other outputs of the
rate controller.  The integrator is limited by the :ref:`ATC_RAT_RLL_IMAX<ATC_RAT_RLL_IMAX__AC_AttitudeControl_Heli>` in roll and
:ref:`ATC_RAT_PIT_IMAX<ATC_RAT_PIT_IMAX__AC_AttitudeControl_Heli>` in pitch.  When ground speed is less than 5 m/s, the
integrator is leaked off (reduced at a specified rate) and another parameter, 
:ref:`ATC_RAT_RLL_ILMI<ATC_RAT_RLL_ILMI>` and :ref:`ATC_RAT_PIT_ILMI <ATC_RAT_PIT_ILMI>`, only lets it leak off so much.  If the 
ILMI, or integrator leak minimum, is zero then the integrator will not be 
allowed to grow and the attitude will not be driven to exactly match the 
software’s predicted attitude.  However, if this is non zero or large enough for
attitude errors that may be encountered at low speeds and in a hover, then the 
actual attitude will track the predicted attitude. The reason for the leak and 
ILMI parameter is that a larger amount of integrator is needed for forward 
flight. However, in a hover and in particular during air ground transition, 
allowing large amounts of integrator can cause the aircraft to flip itself on
its side.  So the integrator leak along with the leak minimum parameter keep 
enough of the integrator to make it effective in keeping the attitudes matching
but not so powerful to cause the aircraft to roll over.

Advanced Tuning for Hover Trim, Loiter Flight Mode and Waypoint Flying
======================================================================
At this point you should have a helicopter that is responsive and yet stable.
But we need to trim the helicopter so it hovers pretty much hands-off in
Stabilize flight mode. And adjust the I-gains for Auto flight mode so it tracks
attitude properly under full autopilot control.

Hover Trim
----------
Trimming the helicopter in pitch and roll axes is an important step to keep the
aircraft from drifting in modes like Stabilize and Althold.  The trim attitude 
in the roll axis is affected by the tail rotor thrust.  All conventional single-
rotor helicopters with a torque-compensating tail rotor hover either right skid 
low or left skid low, depending on which way the main rotor turns. The 
ArduCopter software has a parameter, :ref:`ATC_HOVR_ROL_TRM<ATC_HOVR_ROL_TRM>`, to compensate for this 
phenomenon. Longitudinal CG location will affect the trim attitude in the pitch
axis.  There is no parameter to tell the autopilot what pitch attitude 
the aircraft hovers with no drift. It always targets zero deg pitch as measured
by the autopilot. Therefore the actual pitch attitude the aircraft 
hovers may be 5 deg nose high but the autopilot AHRS Trim value is set
to make it think the attitude is zero deg. 

In order to trim the aircraft, set the :ref:`ATC_HOVR_ROL_TRM<ATC_HOVR_ROL_TRM>` parameter to zero. 
During the initial setup of the autopilot, the ``AHRS_TRIM_x`` values are set 
during the accelerometer calibration on the last step that has you level the 
aircraft. For that step you should have made certain that the shaft was 
perfectly straight up in pitch and roll. For this trim procedure, it is 
recommended that you check it and using the method below.

Measure the actual frame angle (on a portion of the frame that is perpendicular
to the mainshaft) in pitch and roll with your digital pitch gauge. Connected to
your ground station software with MavLink, note the pitch and roll angle the
autopilot is "seeing". Adjust the :ref:`AHRS_TRIM_X<AHRS_TRIM_X>` and :ref:`AHRS_TRIM_Y<AHRS_TRIM_Y>` values so
the autopilot "sees" the identical frame angle you measured with the
digital pitch gauge. You can use the Level Horizon function in your ground station
to level the horizon with the helicopter at actual level. That function will
make the adjustments to the AHRS_TRIM's for you.

The above is necessary so we can accurately measure the roll angle to set the
:ref:`ATC_HOVR_ROL_TRM<ATC_HOVR_ROL_TRM>`. The autopilot now "knows" when the mainshaft is
perfectly vertical.

Load the helicopter with its normal payload, and hover the helicopter
in no-wind conditions in Stabilize flight mode. Land it and pull the log, noting
the roll angle that you had to hold with the stick to keep the helicopter from
drifting. Enter this value in the :ref:`ATC_HOVR_ROL_TRM<ATC_HOVR_ROL_TRM>` parameter in centidegrees.
For a CW turning main rotor if it took 3.5 degrees of right roll to compensate,
enter 350. Negative values are for a CCW turning main rotor that requires left
roll to compensate.

**Important Note** - do not use the radio trims at all. Make sure they are
centered. 

After setting the :ref:`ATC_HOVR_ROL_TRM<ATC_HOVR_ROL_TRM>` now hover the helicopter again. If it still
drifts make small adjustments to the :ref:`SERVO1_TRIM<SERVO1_TRIM>` , :ref:`SERVO2_TRIM<SERVO2_TRIM>` and :ref:`SERVO3_TRIM<SERVO3_TRIM>` .
The chances of getting the swashplate perfectly level during bench setup is very
low and this dynamic tuning is needed to trim the helicopter. If it requires
large deviation from your original ``SERVOx_TRIM`` values it is likely you have a CG
problem, or your initial setup when leveling the swashplate was not very
accurate.

Your helicopter is now trimmed properly. This trimming procedure makes the
difference between a helicopter that is difficult to handle vs one that flies
with true scale quality and handling. 

Adjusting I-gains For High-Speed Autonomous Flight
--------------------------------------------------
Prepare a mission with your ground station software that will fly the 
helicopter, preferably in a figure-8 pattern to make both right and left turns,
at a speed of 6 m/s. Fly the helicopter on this mission, pull the logs from the
microSD card and look at the AHRS desired vs actual pitch, roll and yaw
attitudes in dynamic flight. They should track within 1-2 degrees. If they do
not, increase the ``ATC_RAT_xxx_I`` value for that axis until they do.

Now, fly the same mission, but at higher speed of 9-10 m/s, and analyze the logs
the same way. Make further adjustments to the I-gains and IMAX values as
required. It is not clear what I-gain values will be required as no two
helicopters are the same. But I-gain values from 0.25 - 0.38 are common in pitch
and roll, and 0.18 - 0.30 in yaw. IMAX values of 0.40 - 0.45 are common, however
refer to the 'Setting the I gain, IMAX, and ILMI' section on how to determine
what the IMAX value should be.
