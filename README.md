# EEC136AB-Winter2023-Delta-SolarRover
#include "project.h"
#include "stdio.h"

static uint8 data[1] = {0};

void genericEventHandler(uint32_t event, void *eventParameter) 
{
    switch(event)
    {
      case CY_BLE_EVT_STACK_ON:
      case CY_BLE_EVT_GAP_DEVICE_DISCONNECTED:
      {
        Cy_BLE_GAPP_StartAdvertisement(CY_BLE_ADVERTISING_FAST, CY_BLE_PERIPHERAL_CONFIGURATION_0_INDEX);
        break;
      }
      case CY_BLE_EVT_GATT_CONNECT_IND: // Bluetooth connects to the phone
      {
        break;
      }
      case CY_BLE_EVT_GATTS_WRITE_CMD_REQ:
       {
        cy_stc_ble_gatts_write_cmd_req_param_t *writeReqParameter = (cy_stc_ble_gatts_write_cmd_req_param_t *) eventParameter;
            
        if (CY_BLE_DEVICE_INTERFACE_DEVICE_INBOUND_CHAR_HANDLE == writeReqParameter->handleValPair.attrHandle)
        {
            data[0] = writeReqParameter -> handleValPair.value.val[0];
            Cy_BLE_GATTS_WriteRsp(writeReqParameter->connHandle);
        }
        break;
       }
    }    
}

void bleInterruptNotify() 
{
    Cy_BLE_ProcessEvents();
}


int main(void)
{
    __enable_irq(); // Enable global interrupts
    UART_Start();
    setvbuf(stdin, NULL, _IONBF, 0);
    
    
    // Initialize compare values and start PWM
    int compareValue1 = 70;
    int compareValue2 = 70;
    int compareValue3 = 70;
    int compareValue4 = 70;
    
    PWM_1_Start();
    PWM_2_Start();
    PWM_3_Start();
    PWM_4_Start();
    
    
    // ADC and battery percentage initialization
    ADC_1_Start();
    Cy_SAR_StartConvert(SAR, CY_SAR_START_CONVERT_CONTINUOUS);
    float battery_value = Cy_SAR_GetResult16(SAR, 0);
    float battery_volts = Cy_SAR_CountsTo_mVolts(SAR, 0, battery_value);
    float used_capacity = 0;
    float max_capacity = 2000;
    float battery_percentage = 0;
    
    
    /* Enable CM4.  CY_CORTEX_M4_APPL_ADDR must be updated if CM4 memory layout is changed. */
    Cy_SysEnableCM4(CY_CORTEX_M4_APPL_ADDR); 

    /* Place your initialization/startup code here (e.g. MyInst_Start()) */
    Cy_BLE_Start(genericEventHandler);
    
    
    while (Cy_BLE_GetState() != CY_BLE_STATE_ON)
    {
        Cy_BLE_ProcessEvents();    
    }
    
    Cy_BLE_RegisterAppHostCallback(bleInterruptNotify);
    
    data[0] = 0x06;
   
    
    
    for(;;)
    {
        /* Place your application code here. */
  
        cy_stc_ble_gatt_handle_value_pair_t serviceHandle;
        cy_stc_ble_gatt_value_t serviceData;
        
        serviceData.val = (uint8*)data;
        serviceData.len = 1;
        
        serviceHandle.attrHandle = CY_BLE_DEVICE_INTERFACE_DEVICE_OUTBOUND_CHAR_HANDLE;
        serviceHandle.value = serviceData;
        
        Cy_BLE_GATTS_WriteAttributeValueLocal(&serviceHandle);
        
        // Initialize PWM 
        Cy_TCPWM_PWM_SetCompare0(PWM_1_HW, PWM_1_CNT_NUM, compareValue1);
        Cy_TCPWM_PWM_SetCompare0(PWM_2_HW, PWM_2_CNT_NUM, compareValue2);
        Cy_TCPWM_PWM_SetCompare0(PWM_3_HW, PWM_3_CNT_NUM, compareValue3);
        Cy_TCPWM_PWM_SetCompare0(PWM_4_HW, PWM_4_CNT_NUM, compareValue4);
        
       
        if (data[0] == 0x0) // Idle state
        {
            Cy_GPIO_Write(P5_6_PORT, P5_6_NUM, 0); // Set standby to low: Stop 
        }
        else // Moving
        {
            if (data[0] < 0x07) // The motors are independent to the LEDS and robotic arm
                Cy_GPIO_Write(P5_6_PORT, P5_6_NUM, 1); // Set standby to high: Run
            
            
        if (data[0]  == 0x01) // Move forward
        {
            // Speed of left motor
            compareValue1 = 100;
            // Speed of right motor
            compareValue2 = compareValue1;
            
            // Left motor
            Cy_GPIO_Write(P5_3_PORT, P5_3_NUM, 0);
            Cy_GPIO_Write(P5_4_PORT, P5_4_NUM, 1);
            
            // Right motor 
            Cy_GPIO_Write(P6_2_PORT,P6_2_NUM, 0);
            Cy_GPIO_Write(P6_3_PORT,P6_3_NUM, 1);
            
           // CyDelay(20);
        }
        else if (data[0] == 0x02) // Move backward
        {
            // Speed of left motor
            compareValue1 = 70;
            // Speed of right motor
            compareValue2 = compareValue1;
            
            // Left motor
            Cy_GPIO_Write(P5_3_PORT, P5_3_NUM, 1);
            Cy_GPIO_Write(P5_4_PORT, P5_4_NUM, 0);
            
            // Right motor 
            Cy_GPIO_Write(P6_2_PORT,P6_2_NUM, 1);
            Cy_GPIO_Write(P6_3_PORT,P6_3_NUM, 0);
            
            // Turn on back headlights
            Cy_GPIO_Write(P10_2_PORT,P10_2_NUM, 1);
        }
        else if (data[0] == 0x03) // Rotate left
        {
             // Speed of left motor
            compareValue1 = 50;
            // Speed of right motor
            compareValue2 = compareValue1;
            
            // Left motor
            Cy_GPIO_Write(P5_3_PORT, P5_3_NUM, 1);
            Cy_GPIO_Write(P5_4_PORT, P5_4_NUM, 0);
            
            // Right motor 
            Cy_GPIO_Write(P6_2_PORT,P6_2_NUM, 0);
            Cy_GPIO_Write(P6_3_PORT,P6_3_NUM, 1);
        }
        else if (data[0] == 0x04) // Rotate right
        {
             // Speed of left motor
            compareValue1 = 50;
            // Speed of right motor
            compareValue2 = compareValue1;
            
            // Left motor
            Cy_GPIO_Write(P5_3_PORT, P5_3_NUM, 0);
            Cy_GPIO_Write(P5_4_PORT, P5_4_NUM, 1);
            
            // Right motor 
            Cy_GPIO_Write(P6_2_PORT,P6_2_NUM, 1);
            Cy_GPIO_Write(P6_3_PORT,P6_3_NUM, 0);
        }
        else if (data[0] == 0x05) // Increase motor speed by 10% 
        {  
            if (compareValue1 < 100)
            {
                // Speed of left motor
                compareValue1 = compareValue1 + 10;
                // Speed of right motor
                compareValue2 = compareValue1;
            }
            data[0] = 0x12;
        }
        else if (data[0] == 0x06) // Decrease motor speed by 10%
        {
            if (compareValue1 > 0)
            {
                // Speed of left motor
                compareValue1 = compareValue1 - 10;
                // Speed of right motor
                compareValue2 = compareValue1;
             }
            
            data[0] = 0x12;
        }
        else if (data[0] == 0x12) // Stop increasing speed 
        {
            // Speed of left motor
            compareValue1 = compareValue1;
            // Speed of right motor
            compareValue2 = compareValue2;
        }
        }
        
        if (data[0] == 0x0f) // Grab the object with the robotic arm
        {
            Cy_GPIO_Write(P10_6_PORT, P10_6_NUM, 1); // Set standby2 to high: Turn on robotic arm
            
            // Speed of robot motor
            compareValue3 = 100; 
            compareValue4 = 100;
            
             // Left Robot motor
            Cy_GPIO_Write(P9_4_PORT, P9_4_NUM, 0);
            Cy_GPIO_Write(P9_5_PORT, P9_5_NUM, 1);
            
            // Right Robot motor
            Cy_GPIO_Write(P10_4_PORT, P10_4_NUM, 0);
            Cy_GPIO_Write(P10_5_PORT, P10_5_NUM, 1);
           
        }
        else if (data[0] == 0x10) // Release the object with the robotic arm
        {
            Cy_GPIO_Write(P10_6_PORT, P10_6_NUM, 1); // Set standby2 to high: Turn on robotic arm
            
            // Speed of robot motor
            compareValue3 = 70; 
            compareValue4 = 70;
            
             // Left Robot motor
            Cy_GPIO_Write(P9_4_PORT, P9_4_NUM, 1);
            Cy_GPIO_Write(P9_5_PORT, P9_5_NUM, 0);
            
            // Right Robot motor
            Cy_GPIO_Write(P10_4_PORT, P10_4_NUM, 1);
            Cy_GPIO_Write(P10_5_PORT, P10_5_NUM, 0);
              
        }
        else if (data[0] == 0x11)
        {
            Cy_GPIO_Write(P10_6_PORT, P10_6_NUM, 0); // Set standby2 to low: Turn off robotic arm
        }
        
        if (data[0] == 0x07) // turn on front headlights
        {
            Cy_GPIO_Write(P9_1_PORT,P9_1_NUM, 1);
        }
        else if (data[0] == 0x08) // turn off front headlights
        {
            Cy_GPIO_Write(P9_1_PORT,P9_1_NUM, 0);
        }
        
        if (data[0] == 0x09) // turn on back headlights
        {
            Cy_GPIO_Write(P10_2_PORT,P10_2_NUM, 1);
        }
        else if (data[0] == 0x0a) // turn off back headlights
        {
            Cy_GPIO_Write(P10_2_PORT,P10_2_NUM, 0);
        }
        
        if (data[0] == 0x0b) // turn on headlight indicator to turn right
        {
            Cy_GPIO_Write(P9_2_PORT,P9_2_NUM, 1);
            CyDelay(1000);
            Cy_GPIO_Write(P9_2_PORT,P9_2_NUM, 0);
            CyDelay(1000);
        }
        else if (data[0] == 0x0c) // turn off the right headlight indicator
        {
            Cy_GPIO_Write(P9_2_PORT,P9_2_NUM, 0);
        }
        
        if (data[0] == 0x0d) // turn on headlight indicator to turn left
        {
            Cy_GPIO_Write(P9_3_PORT,P9_3_NUM, 1);
            CyDelay(1000);
            Cy_GPIO_Write(P9_3_PORT,P9_3_NUM, 0);
            CyDelay(1000);
        }
        
        else if (data[0] == 0x0e) // turn off the left headlight indicator 
        {
            Cy_GPIO_Write(P9_3_PORT,P9_3_NUM, 0);
        }
        
          if (data[0] != 0x02 & data[0] != 0x09) // turn back headlights off after moving backwards
        {
            Cy_GPIO_Write(P10_2_PORT,P10_2_NUM, 0);
        }
        
        
        // Battery percentage determination
        battery_value = Cy_SAR_GetResult16(SAR, 0);
        battery_volts = Cy_SAR_CountsTo_mVolts(SAR, 0, battery_value);
        
        
        if (battery_volts > 3700)
        {
            used_capacity = (4000 - battery_volts) / 0.375;
        }
        else if (3700 >= battery_volts && battery_volts > 3600)
        {
            used_capacity = (3820 - battery_volts) / 0.14;
        }
        else if (3600 >= battery_volts && battery_volts > 3500) 
        {
            used_capacity = (4400 - battery_volts) / 0.5;
        }
        else if (3500 >= battery_volts && battery_volts > 3000) 
        {
            used_capacity = (8000 - battery_volts) / 2.5;    
        }
        else if (battery_volts < 3000)
        {
            used_capacity = max_capacity;    
        }
        else 
        {
            used_capacity = max_capacity;  
        }
        
        battery_percentage = (1 - (used_capacity / max_capacity)) * 100;
        
       // printf("%f ", battery_volts);
       // printf("%f ", battery_percentage);
        
        CyDelay(20);
        
    }
}

/* [] END OF FILE */
