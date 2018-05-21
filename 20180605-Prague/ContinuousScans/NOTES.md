# Introduction

# Motion

1. Limits protection - example
    * `Door> set_lim mot01 0`
    * `Door> ascanct mot01 0 10 10 0.1`
    * Error: motor pre-start position -10
    * Position software limits are verified before the scan against the
      pre-start and post-end positions.
2. Motion trajectory - slide:
    * For each physical motor there are 4 positions in the scan: pre-start,
      start, post-end and end.
    * All motors are started at the same time.
    * All motors have the same acceleration and deceleration time during the
      scan - the one of the slowest accelerating/decelerating motor.
    * Scan velocity is calculated for each of the motors taking into account the
      integration and latency times and the motor's displacement
3. Maximum velocity - example:
    * mot01 maximum velocity limit set to 100 (jive)
    * `$> taurustrend mot01/position`
    * `Door> ascanct mot01 0 10 10 1`
    * All motors goes to the pre-start position with the maximum velocity (if
      defined).
    * All motors corrects the overshoot (go to the end position) with
      the maximum velocity (if defined)
4. Parameters backup - example:
    * `$> taurustrend mot01/position`
    * `Door> mot01.velocity = 10`
    * `Door> mv mot01 30`
    * `Door> ascanct mot01 0 10 10 0.1`
    * Motors attributes (acceleration, deceleration and velocity) are backup
      before any modification.
    * Backup motor attributes are restarted after the scan.
5. Pseudo motors scan - example: 
    * `$> taurustrend mot01/position mot02/position gap01/position`    
    * `Door> ascanct gap01 0 10 10 0.1`
    * Even if the user asked for the scan of the pseudo motor the scan is
      executed on the physical motors always - all of them move on the linear
      trajectory.
6. Other topics:
    * Motion controller may not be able to move at this velocity and can silently
      adjust its value to the closest possible velocity (stepper motors have
      discrete motor resolution and discrete frequency) causing desynchronization
      of the scan.
    * Motor position updates are used as the base for the software
      synchronization (see `MotionLoop_StatesPerPosition` and
      `MotionLoop_SleepTime`) - big amount of motor groups and pseudo motors may
      affect the position updates freqeuncy as well.
    * Motor which does not reach the final position will cause the software
      synchronizer wait infinitelly - watchdog mechanism (15 seconds timeout)
      prevents that.
7. TODOs:
    * Acceleration attribute is expecting to return acceleration time in seconds.
    * Motion calculations done on the pool side, or maybe directly coordinated
      motion of pseudo motors/motor groups?

# Acquisition

1. Software synchronized acquisition - example:
    * `$> taurustrend -r 10 mot01/position "eval:bool({ct01/state})"`
    * `Door> defmeas cs-test ct01 ct12`
    * `Door> senv ActiveMntGrp cs-test`
    * `Door> ascanct mot01 0 10 10 1`  # skipped acquisitions
    * Software synchronized acquisitions may be skipped when:
        * previous acquisition is still in progress
        * the position/time has passed
    * Software synchronized channels are loaded multiple times for single
      acquisition
    * Software synchronized channels returns only one value.
2. Configuration - expconf:
    * Measurement group contains configuration attribute - list of
      channels/controllers and their synchronization configuration:
        * synchronizer - either a TriggerGate element or software synchronizer
        * synchronization - Trigger/Gate
    * Synchronizer and synchronization are configured and loaded on the controller
      level and not on the channel level.
    * Controller is loaded with the synchronization type: SoftwareTrigger, 
      HardwareTrigger, SoftwareGate or HardwareGate
3. Coordination and actions - slide:
    * Acquisition is managed by the measurement group and acquisition 
      actions (threads).
    * Based on the configuration channels are assigned to particular acquisition
      actions: software, hardware, 0D.
4. Hardware synchronized acquisition - example:
    * `$> taurustrend -r 10 mot01/position "eval:bool({ct01/state})" 
      "eval:bool({ct12/state})"`
    * `Door> expconf` # change synchronizer for ct11 to dummy trigger/gate tg01
    * Hardware synchronized channels are loaded only once with the number of
      acquisitions
    * Hardware synchronized channels may return multiple values a.k.a. chunks.
5. Latency time - slide:
    * `$> taurustrend -r 10 mot01/position "eval:bool({ct01/state})"`
    * `Door> ascanct mot01 0 10 10 1 0.1`  # all acquisitions
    * Latency time may be forced by the user in order to avoid loosing software
      synchronized acquisitions.
    * Hardware synchronized controller must return proper latency time -
      otherwise the hardware may not have enough time to re-arm for the next
      acquisition.
    * The maximum latency time (between all the controllers and the one
      specified by the user) will be used during the scan.
6. Pseudo counters:  
    * Indexes are assigned to the returned values so they can be merged into
      records.
    * Pseudo counters calculations are based on indexes as well - all values are
      necessary to perform calculations.
    * `Door> expconf`  # add ioveri001 to the measurement group
    * `Door> ascanct mot01 0 10 10 1`  # calculations done only if both 
      values are present 
7. Other topics:
    * No distinction is made for different trigger types: pre-trigger,
      mid-trigger and post-trigger

8. TODOs:
    * Optimize software acquisition - not necessary to load on every point, 
      reducte time overheads
    * Exceptions during the acquisition are just reported with the measurement
      group's fault state - lack of feedback what actually went wrong.
    * Extra attributes (Tango attributes) are not directly supported yet - one
      needs to create a controller instance to point to them.
    * Software gate is not yet implemented.


# Synchronization

1. Software synchronization:
    * Done by an internal software synchronizer  object (thread) - one per 
      measurement group.
    * Emits Active and Passive events. On this events acquisition callbacks
      are executed.
    * Callback execution time is considered when generating events in order to 
      avoid drift.
    * Events may be skipped if the position/time has already passed and the 
      next one is more up-to-date.
    * Selects the initial configuration in the position domain and active
      configuration in the time domain.
    * The initial configuration can be forced to use time domain using the 
      measurement group's `SoftwareSynchronizerInitialDomain` attribute.
    * Measurement group's `Moveable` attribute is used to set the master motor
      which position will be used as the synchronization base.
2. Hardware synchronization:
    * Done by an external device, could be a motor controller or a timing 
      card e.g. IcePAP, PiLC, Zebra, etc. integrated via TriggerGate 
      controllers (usually requires hardware cabling between this 
      device and the experimental channel controller).
3. Synchronization description - slide:
    * A unique synchronization description, prepared by the scan framework is 
      loaded to both the hardware synchronizers and the software synchronizer.
    * This description contains information in both position domain and time 
      domain (timescan contains only time domain).
    * TriggerGate controller should select the initial configuration in the 
      position domain and active configuration in the time domain - acquisitions 
      should happen at given position and last a given time period.
4. Other topics:
    * Synchronizers are always configured to generate Gate, even if the channels
      are configured to receive Trigger.
    * Synchronizers' states are monitored during the measurement group's
      acquisition and they participate in the overall state calculation.
5. TODOs:
    * Moveable reference should be passed to the TriggerGate controller - 
      necessary to implement trigger multiplexing in IcePAP. 
   

# Data
1. Communication channel:
    * Are passed via `Data` (to be renamed to `ValueBuffer`) attribute Tango 
      events - JSON serialized pairs of values and indexes.
    * Event callbacks are executed by a single worker thread in the MacroServer 
      to avoid blocking the Tango event consumer thread.
2. Data merging
    * Data from different experimental channels may arrive in an arbitrary order
      but from the same experimental channel always come ordered (but may not 
      be consecutive - missed acquisitions).
    * `Door> ascanct mot01 0 10 10 1`  #  skipped acquisitions
    * `Door> senv ApplyInterpolation True`
    * `Door> ascanct mot01 0 10 10 1`  #  interpolated values
    * Missing data may be extrapolated and interpolated (zero-order 
      interpolation) - `ApplyExtrapolation` and `ApplyInterpolation` environment
      variables.
3. TODOs:
    * Pass timestamps with the data.


# Framework
* `ascanc` macros provides desynchronized, in the sense of number of 
acquisition, scans.
* `aNscanct`, `dNscanct`, `meshct` and `timescan` use the synchronized
approach.
* Several hook places are available for custom custom configuration:
  * `pre-scan`
  * `pre-configuration`
  * `post-configuration`
  * `pre-start`
  * `pre-cleaunp`
  * `post-cleanup`
  * `post-scan`

TODOs:
* Provide an easy way of developing custom scans - waypoint generator with 
  a custom synchronization description.
* Merge both `-c` and `-ct` like scans in one
