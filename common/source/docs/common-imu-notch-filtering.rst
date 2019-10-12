.. _common-imu-notch-filtering:

==========================================
Managing Gyro Noise with the Static Notch and Dynamic Harmonic Notch Filters
==========================================

As :ref:`discussed<common-vibration-damping>`, managing vibration in ardupilot flight controller installations is extremely important in order to yield predictable control of an aircraft. Typically installations utilise mechanical vibration damping in order to remove the worst of the vibration. However, mechanical damping can only go so far and software filtering must be used to remove further noise. To the flight controller, vibration noise looks like any other disturbance (e.g. wind) that the flight controller must compensate for in order to control the aircraft. Ardupilot uses a :ref:`low-pass <copter:INS_GYRO_FILT>` software filter to remove much of this remaining vibration noise, however, filtering has an unwanted side-effect - it removes *everything* including attitudinal and control information about the aircraft. Thus, while aggressive filtering can eliminate noise it can also reduce the control and responsiveness of the aircraft. This problem gets particularly acute on aircraft with very high levels of low-frequency noise (e.g. helis) and those with low-levels of natural mechanical damping that require greater control (e.g. small, high-powered copters).

Virtually all vibrations originate from the motors and, importantly, we actually know quite a lot about this noise source as the majority of it is linked to the motor rotational frequency. It is thus theoretically possible to construct a software filter that targets *just* this noise, leaving all of the useful gyro information alone. This is what a *notch filter* does - it targets a narrow band of frequencies.

Ardupilot has support for two different notch filters - a static notch filter that can be set at a fixed frequency and a dynamic notch filter that can be targeted at a range linked to the motor rotational frequency.

With the introduction of dynamic notch filtering, the need for static notch filtering is reduced, although it can be useful in targeting particular resonant frequencies (e.g. of the frame). We will therefore look at dynamic filtering first.

Pre-Flight Setup
================

In order to configure the dynamic harmonic notch filter it is important to establish a baseline that identifies the motor noise at the hover throttle level. To do this we need to use the :ref:`batch sampler<common-imu-batchsampling>`

- Set :ref:`INS_LOG_BAT_MASK <INS_LOG_BAT_MASK>` = 1 to collect data from the first IMU
- :ref:`LOG_BITMASK <copter:LOG_BITMASK>`'s IMU_RAW bit must **not** be checked.  The default LOG_BITMASK value is fine
- Set :ref:`INS_LOG_BAT_OPT <INS_LOG_BAT_OPT>` = 0 to capture pre-filter gyro data

Flight and Post-Flight Analysis
===============================

- Perform a hover flight of at least 30s in altitude hold and :ref:`download the dataflash logs <common-downloading-and-analyzing-data-logs-in-mission-planner>`
- Open Mission Planner, press Ctrl-F, press the FFT button, press "new DF log" and select the .bin log file downloaded above

.. image:: ../../../images/imu-batchsampling-fft-mp2.png
    :target:  ../_images/imu-batchsampling-fft-mp2.png
    :width: 450px

On the graph it should be possible to identify a significant peak in noise that corresponds to the motor rotational frequency. On a smaller copter this is likely to be around 200Hz and on a larger copter 100Hz or so. On a traditional helicopter this can be as low as 25Hz.

- With the same log, open it in the regular way in mission planner and graph the throttle value. From this identify an average hover throttle value. It's also possible to use :ref:`MOT_HOVER_LEARN <MOT_HOVER_LEARN>` = 2 and read off the value of :ref:`MOT_THST_HOVER <MOT_THST_HOVER>`

- This gives you a hover motor frequency *hover_freq* and thrust value *hover_thrust*

Harmonic Notch Configuration
============================

- Set :ref:`INS_HNTCH_ENABLE <INS_HNTCH_ENABLE>` = 1 to enable the harmonic notch
- Set :ref:`INS_HNTCH_REF <INS_HNTCH_REF>` = *hover_thrust* to set the harmonic notch reference value
- Set :ref:`INS_HNTCH_FREQ <INS_HNTCH_FREQ>` = *hover_freq* to set the harmonic notch reference frequency
- Set :ref:`INS_HNTCH_BW <INS_HNTCH_BW>` = *hover_freq* / 2 to set the harmonic notch bandwidth

Post Configuration Flight and Post-Flight Analysis
===============================

- This time set :ref:`INS_LOG_BAT_OPT <INS_LOG_BAT_OPT>` = 2 to capture post-filter gyro data

Perform a similar hover flight and analyze the dataflash logs in the same way. This time you should see significantly less noise and, more significantly, attenuation of the motor noise peak. If the peak does not seem well attenuated then you can experiment with increasing the bandwidth and attenuation of the notch. However, the wider the notch the more delay it will introduce into the control of the aircraft so doing this can be counter-productive.

Notch Frequency Scaling
=======================

The harmonic notch is designed to match the motor noise frequency as it changes by interpreting the throttle value. The frequency is scaled up from the hover frequency and will never go below this frequency. However, in dynamic flight it is quite common to hit quite low motor frequencies during propwash. In order to address this it is possible to change the ref value in order to scale from a lower frequency.

- First perform a long dynamic flight using your current settings and post-filter batch logging. Examine the FFT and look at how far the motor noise peak extends below the hover frequency. Use this frequency - *min_freq* - as the lower bound of your scaling. Then in order to calculate an updated value of the throttle reference use:

:ref:`INS_HNTCH_REF <INS_HNTCH_REF>` = :math:`hover_thrust * \sqrt{min_freq / hover_freq}`
