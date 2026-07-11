odrv0.erase_configuration()
odrv0.config.dc_max_positive_current = 24
odrv0.config.dc_max_negative_current = -5.0
odrv0.axis0.motor.config.motor_type = MOTOR_TYPE_GIMBAL 
odrv0.axis0.motor.config.pole_pairs = 7                 
odrv0.axis0.motor.config.calibration_current =5     
odrv0.axis0.encoder.config.mode = ENCODER_MODE_INCREMENTAL                      
odrv0.axis0.encoder.config.cpr = 4096                                
odrv0.axis0.controller.config.control_mode = CONTROL_MODE_VELOCITY_CONTROL 
odrv0.axis0.controller.config.vel_gain = 0.02                         
odrv0.axis0.controller.config.vel_integrator_gain = 0.5               
odrv0.axis0.controller.config.vel_limit = 10                          
odrv0.axis0.controller.config.input_mode = INPUT_MODE_VEL_RAMP      
odrv0.axis0.controller.config.vel_ramp_rate = 10         

odrv0.axis1.motor.config.motor_type = MOTOR_TYPE_GIMBAL 
odrv0.axis1.motor.config.pole_pairs = 7                 
odrv0.axis1.motor.config.calibration_current = 5     
odrv0.axis1.encoder.config.mode = ENCODER_MODE_INCREMENTAL                      
odrv0.axis1.encoder.config.cpr = 4096                                
odrv0.axis1.controller.config.control_mode = CONTROL_MODE_VELOCITY_CONTROL 
odrv0.axis1.controller.config.vel_gain = 0.02                         
odrv0.axis1.controller.config.vel_integrator_gain = 0.5               
odrv0.axis1.controller.config.vel_limit = 10                          
odrv0.axis1.controller.config.input_mode = INPUT_MODE_VEL_RAMP      
odrv0.axis1.controller.config.vel_ramp_rate = 10   

odrv0.save_configuration()
odrv0.reboot()

odrv0.axis0.requested_state = AXIS_STATE_FULL_CALIBRATION_SEQUENCE
odrv0.axis0.error                                     

odrv0.axis1.requested_state = AXIS_STATE_FULL_CALIBRATION_SEQUENCE
odrv0.axis1.error 

odrv0.axis0.requested_state = AXIS_STATE_CLOSED_LOOP_CONTROL
odrv0.axis1.requested_state = AXIS_STATE_CLOSED_LOOP_CONTROL

odrv0.axis0.controller.input_vel = 5 
odrv0.axis1.controller.input_vel = 5 

odrv0.axis0.motor.config.pre_calibrated = True        
odrv0.axis0.encoder.config.pre_calibrated = True    
odrv0.axis0.config.startup_closed_loop_control = True   


odrv0.axis1.motor.config.pre_calibrated = True        
odrv0.axis1.encoder.config.pre_calibrated = True    
odrv0.axis1.config.startup_closed_loop_control = True 


odrv0.save_configuration()
odrv0.reboot()



======================================================

import time

odrv0.erase_configuration()
odrv0.config.dc_max_positive_current = 24
odrv0.config.dc_max_negative_current = -5.0

odrv0.axis0.motor.config.motor_type = MOTOR_TYPE_GIMBAL
odrv0.axis0.motor.config.pole_pairs = 7
odrv0.axis0.motor.config.calibration_current = 1.0          
odrv0.axis0.motor.config.resistance_calib_max_voltage = 14.0 
odrv0.axis0.encoder.config.mode = ENCODER_MODE_INCREMENTAL
odrv0.axis0.encoder.config.cpr = 4096
odrv0.axis0.encoder.config.use_index = True                 
odrv0.axis0.controller.config.control_mode = CONTROL_MODE_VELOCITY_CONTROL
odrv0.axis0.controller.config.vel_gain = 0.02
odrv0.axis0.controller.config.vel_integrator_gain = 0.5
odrv0.axis0.controller.config.vel_limit = 10
odrv0.axis0.controller.config.input_mode = INPUT_MODE_VEL_RAMP
odrv0.axis0.controller.config.vel_ramp_rate = 10

odrv0.axis1.motor.config.motor_type = MOTOR_TYPE_GIMBAL
odrv0.axis1.motor.config.pole_pairs = 7
odrv0.axis1.motor.config.calibration_current = 1.0
odrv0.axis1.motor.config.resistance_calib_max_voltage = 14.0
odrv0.axis1.encoder.config.mode = ENCODER_MODE_INCREMENTAL
odrv0.axis1.encoder.config.cpr = 4096
odrv0.axis1.encoder.config.use_index = True
odrv0.axis1.controller.config.control_mode = CONTROL_MODE_VELOCITY_CONTROL
odrv0.axis1.controller.config.vel_gain = 0.02
odrv0.axis1.controller.config.vel_integrator_gain = 0.5
odrv0.axis1.controller.config.vel_limit = 10
odrv0.axis1.controller.config.input_mode = INPUT_MODE_VEL_RAMP
odrv0.axis1.controller.config.vel_ramp_rate = 10

odrv0.save_configuration()
time.sleep(5.0)

odrv0.axis0.requested_state = AXIS_STATE_FULL_CALIBRATION_SEQUENCE
time.sleep(15.0)  

odrv0.axis1.requested_state = AXIS_STATE_FULL_CALIBRATION_SEQUENCE
time.sleep(15.0) 

dump_errors(odrv0)

odrv0.axis0.motor.config.pre_calibrated = True
odrv0.axis0.encoder.config.pre_calibrated = True
odrv0.axis1.motor.config.pre_calibrated = True
odrv0.axis1.encoder.config.pre_calibrated = True

odrv0.axis0.config.startup_encoder_index_search = True
odrv0.axis0.config.startup_closed_loop_control = True
odrv0.axis1.config.startup_encoder_index_search = True
odrv0.axis1.config.startup_closed_loop_control = True

odrv0.save_configuration()
time.sleep(5.0)

odrv0.axis0.requested_state = AXIS_STATE_CLOSED_LOOP_CONTROL
odrv0.axis1.requested_state = AXIS_STATE_CLOSED_LOOP_CONTROL
time.sleep(1.0)

odrv0.axis0.controller.input_vel = 5.0
odrv0.axis1.controller.input_vel = 5.0
time.sleep(6.0)

odrv0.axis0.controller.input_vel = 0.0
odrv0.axis1.controller.input_vel = 0.0
odrv0.axis0.requested_state = AXIS_STATE_IDLE
odrv0.axis1.requested_state = AXIS_STATE_IDLE



odrv0.axis0.controller.config.control_mode = CONTROL_MODE_POSITION_CONTROL
odrv0.axis1.controller.config.control_mode = CONTROL_MODE_POSITION_CONTROL


odrv0.axis0.controller.input_pos = 5.0
odrv0.axis1.controller.input_pos = 5.0

odrv0.axis0.motor.config.direction = -1 
odrv0.axis1.motor.config.direction = -1