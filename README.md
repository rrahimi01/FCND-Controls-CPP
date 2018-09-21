# FCND-Controls-CPP
Udacity Flying Car Controls Assignment 
Date: 24 Sept 2018



![alt text](/pics/controllers_overview.png "controllers")


## Intro

This step was to tune the mass so that the quadcopter dowsnt fall to the ground. I tuned to the following number:

Mass = 0.4855



## Body rate control

Requirements: The controller should be a proportional controller on body rates to commanded moments. The controller should take into account the moments of inertia of the drone when calculating the commanded moments.

There are two funcitons to configure in this step, GenerateMotorCommands() and BodyRateControl()

### GenerateMotorCommands()

This funciton calculates the thrust per rotor based on the total given thrust and momentum and saves the values in cmd.desiredThrustsN array.

First step is to convert the rotor-to-rotor length that is given to a prependicular distance from the rotor to the axes using the formula below: 

length = L / (2 * sqrtf(2))

The total momentum must then be devided into force applied on the rotors on X, Y and Z directions and values are saved in f_p , f_q and f_r variables for force applied about x, y and z directions. The calculated f_p, f_q and f_r forces are then added to each rotor with a negative or possitive value, based on its position and rotation of the rotors described below: 


- For roll motion, the moments generated by the two left propellers are counteracted by the moment generated by the two propellers on the right of the body meaning: 
    
    (front_left_rotor + rear_left_rotor − front_right_rotor − rear_right_rotor)

- Similarly, the pitch is generated by the mismatch of the moments created by front propellers and the moment generated by the rear propellers on the multi-rotor's bodyframe:

    (front_left_rotor + front_right_rotor − rear_left_rotor − rear_right_rotor)


- And the yaw motion is executed by the mismatch of the moments generated by the propellers along the z axis by the reactive force. The moment generated by the propeller is directed opposite of its rotation and is proportional to the square of the angular velocities. Clockwise rotation produces moment in counterclockwise direction with a positive value. Counter-clockwise rotation produces clockwise moment and we give it a negative value. 

    (front_left_rotor - front_right_rotor − rear_left_rotor + rear_right_rotor)


Below follows the code:


  
    VehicleCommand QuadControl::GenerateMotorCommands(float collThrustCmd, V3F momentCmd)
    {
        //L = full rotor to rotor distance that is defined in QuadControlParams.txt
        // I need to calculate the perpendicular distance to axes that is:
        float length = L / (2.f * sqrtf(2.f)); //perpendicular distance to axes
    
        // Calculating the moments created by the propellers
        // kappa is drag/thrust ratio
        float f_total = collThrustCmd;      // (front_left_rotor + front_right_rotor + rear_left_rotor + rear_right_rotor)
        float f_p = momentCmd.x / length;   // (front_left_rotor + rear_left_rotor − front_right_rotor − rear_right_rotor)
        float f_q = momentCmd.y / length;   // (front_left_rotor + front_right_rotor − rear_left_rotor − rear_right_rotor)
        float f_r = -momentCmd.z / kappa;   // (front_left_rotor - front_right_rotor − rear_left_rotor + rear_right_rotor) NOTE: Reversing the direction becouse the z conrdinate upp is negative.
    
    
        cmd.desiredThrustsN[0] = (f_total + f_p + f_q + f_r) / 4.f; // front_left, CW rotation,
        cmd.desiredThrustsN[1] = (f_total - f_p + f_q - f_r) / 4.f; // front_right, CCW rotation
        cmd.desiredThrustsN[2] = (f_total + f_p - f_q - f_r) / 4.f; // rear_left, CW rotation,
        cmd.desiredThrustsN[3] = (f_total - f_p - f_q + f_r) / 4.f; // rear_right, CCW rotation
        return cmd;
    }


### BodyRateControl()

BodyRateControl() is a first order P controller that must be tuned first before any other controller. For this funciton, the desired and actual pqr values are given that we can use to calculate the pqr_error. Using the pqr_error, the moment of inertia and kpPQR constant, we can calculate and return the desired moment for each of the 3-axis. 

After implementing this function, the kpPQR contact must be tuned. in this case, the tuned value is: 

    kpPQR = 41.5, 41.5, 6


    V3F QuadControl::BodyRateControl(V3F pqrCmd, V3F pqr)
    {
        V3F momentCmd;
        V3F I = V3F(Ixx, Iyy, Izz);     //Converting to V3F object
        V3F pqr_error = pqrCmd - pqr;   //"desired body rates" - "current or estimated body rates"
        momentCmd = I * kpPQR * ( pqr_error ); //return a V3F containing the desired moments for each of the 3 axes
        return momentCmd;
    }


In QuadControlParams.txt, kpPQR is (41.5, 41.5, 5)

Result:

- PASS: ABS(Quad.Roll) was less than 0.025000 for at least 0.750000 seconds
- PASS: ABS(Quad.Omega.X) was less than 2.500000 for at least 0.750000 seconds




## Roll pitch control

Requirements: The controller should use the acceleration and thrust commands, in addition to the vehicle attitude to output a body rate command. The controller should account for the non-linear transformation from local accelerations to body rates. Note that the drone's mass should be accounted for when calculating the target angles.


### RollPitchControl()

RollPitchControl() provides the desired acceleration in global cordinates, the estimated attitude and desired collective trhust. In return, we need to calculate and return the desired pith and roll rates of the vehacle. 

This is the second P controller that we need to tune, and it is a first order system. After implementing this function, the kpBank constant needs to be tuned. Below follows the result in this scenario.

    kpBank = 13

    Result:
    - PASS: ABS(Quad.Roll) was less than 0.025000 for at least 0.750000 seconds
    - PASS: ABS(Quad.Omega.X) was less than 2.500000 for at least 0.750000 seconds


The calculations needed some conversion between local body frame and world cordinates. Below follows the implemented function.


    V3F QuadControl::RollPitchControl(V3F accelCmd, Quaternion<float> attitude, float collThrustCmd)
    {
      V3F pqrCmd;
      Mat3x3F R = attitude.RotationMatrix_IwrtB();
      
      if ( collThrustCmd > 0 )
      {

            float c = - collThrustCmd / mass;

            // R13 (target X) & R23 (target Y)
            float x_c_target = CONSTRAIN(accelCmd.x / c, -maxTiltAngle, maxTiltAngle);
            float y_c_target = CONSTRAIN(accelCmd.y / c, -maxTiltAngle, maxTiltAngle);

            float x = R(0,2); // R13 (actual X)
            float x_err = x_c_target - x;
            float x_p_term = kpBank * x_err;

            float y = R(1,2); // R23 (Actual Y)
            float y_err = y_c_target - y;
            float y_p_term = kpBank * y_err;

            pqrCmd.x = (R(1,0) * x_p_term - R(0,0) * y_p_term) / R(2,2);
            pqrCmd.y = (R(1,1) * x_p_term - R(0,1) * y_p_term) / R(2,2);
      }
      else
      {
            pqrCmd.x = 0.0;
            pqrCmd.y = 0.0;
      }
      return pqrCmd;
    }



## Altitude controller

Requirements: The controller should use both the down position and the down velocity to command thrust. Ensure that the output value is indeed thrust (the drone's mass needs to be accounted for) and that the thrust includes the non-linear effects from non-zero roll/pitch angles.

### AltitudeControl()

A second order PD controller that we need to implement and tune after the roll-pitch control. This function provides desired and current vertical positions in NED cordinates, feed-forward vertical acceleration and the timestep. It must calculate and return the collective thrust command. 

Below follows a copy of the function: 

    float QuadControl::AltitudeControl(float posZCmd, float velZCmd, float posZ, float velZ, Quaternion<float> attitude, float accelZCmd, float dt)
    {
        Mat3x3F R = attitude.RotationMatrix_IwrtB();
        float thrust = 0;

        float error = posZCmd - posZ;
        float error_dot = velZCmd - velZ;
        integratedAltitudeError += error * dt;

        float p_term = kpPosZ * error;
        float i_term = KiPosZ * integratedAltitudeError;
        float d_term = kpVelZ * error_dot;

        float b_z = R(2,2);
        float u1_bar = p_term + i_term + d_term + accelZCmd;
        float acc = (u1_bar - CONST_GRAVITY) / b_z;
        acc = CONSTRAIN(acc, - maxAscentRate / dt, maxDescentRate / dt);

        thrust = - mass * acc ;
        return thrust;
    }




## Lateral position control

Requirements: The controller should use the local NE position and velocity to generate a commanded local acceleration.

### LateralPositionControl() 

This function provides desired position and velocity, current position and velocity and the feed-forward acceleration. It calculates and returns desired horizontal acceleration. 

Below follows the implementation: 

    V3F QuadControl::LateralPositionControl(V3F posCmd, V3F velCmd, V3F pos, V3F vel, V3F accelCmdFF)
    {
          accelCmdFF.z = 0;
          velCmd.z = 0;
          posCmd.z = pos.z;

          V3F accelCmd = accelCmdFF;
          V3F pos_err = posCmd - pos ;

          if (velCmd.mag() > maxSpeedXY) 
          {
                velCmd = velCmd.norm() * maxSpeedXY;
          }

          V3F vel_err = velCmd - vel ;

          accelCmd = kpPosXY * pos_err + kpVelXY * vel_err + accelCmd;

          if (accelCmd.mag() > maxAccelXY) 
          {
                accelCmd = accelCmd.norm() * maxAccelXY;
          }

          accelCmd.z = 0;
          return accelCmd;
    }


After the implementation, the following constants must be tuned. With the following values, the tests passed. 

    kpPosXY = 50
    kpPosZ = 40
    KiPosZ = 40
    kpVelXY = 20
    kpVelZ = 14



## Yaw control

Requirements: The controller can be a linear/proportional heading controller to yaw rate commands (non-linear transformation not required).






## Calculating the motor commands given commanded thrust and moments

Requirements: The thrust and moments should be converted to the appropriate 4 different desired thrust forces for the moments. Ensure that the dimensions of the drone are properly accounted for when calculating thrust from moments.




## Evaluation

Requirements: Ensure that in each scenario the drone looks stable and performs the required task. Specifically check that the student's controller is able to handle the non-linearities of scenario 4 (all three drones in the scenario should be able to perform the required task with the same control gains used).





