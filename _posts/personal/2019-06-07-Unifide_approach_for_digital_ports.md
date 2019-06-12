---
layout: post
title: Unifide approach for digital ports
blogid: personal
sticky: false
published: false
---
In recent design there is a requirement for hundreds of GPIO ports on the MCU.
STM32F373 in 100-pin LQFP processor used for GPIO, analogue and resistance measurements is perfect for the task due to Sigma Delta ADC and rich peripherals. But it lacks required count of GPIO ports. In the same time the bigger processor is prohibited by mechanical limitations of the board, production process and logistical limitations.

The solution is to use port expanders. 
My challenge was to integrate shift register port expanders with clear separation of BSP and application layers.

There are two possibilities to toggle a GPIO port:
     via SIPO registers, using bsp module responsible for data transfer. 
     via MCU GPIO, using interfaces provided by CubeMX.
```C
typedef enum hal_gpio_index_e
{
  /* DigitalOutputX means number of the output in circuit diagram. It is
   * different than symbol of the output on the front panel of the device. */
   DigitalOutput1,
   DigitalOutput2,
   Led1,
   Led2,
   UnusedPin1 ,
   UnusedPin2 ,
   NumberOfGPIO
} hal_gpio_index_et;



//!@{
/*!\brief Structure used to initialize microcontroller inputs and outputs. */
typedef struct port_init
{
   GPIO_TypeDef *GPIO_Port; /*!< Port */
   GPIO_InitTypeDef GPIO_InitStructure; /*!< Pin, speed and mode */
   pin_init_value_et PinInitValue; /*!< Initial pin value */
} port_init_t;
//!@}

static const port_init_t PortInit[] = {

/* Index Port Pin Mode Speed Output type Pull-up/Pull down Init value Pin activation state*/

/* Digital outputs of the device with their power supplies and input signals used to check short circuit or overload. */
[DigitalOutput1] = { GPIOC, { GPIO_Pin_2, GPIO_Mode_OUT, GPIO_Speed_10MHz, GPIO_OType_PP, GPIO_PuPd_NOPULL }, PinLow },
[DigitalOutput2] = { GPIOA, { GPIO_Pin_5, GPIO_Mode_OUT, GPIO_Speed_10MHz, GPIO_OType_PP, GPIO_PuPd_NOPULL }, PinLow },

[Led1] = { GPIOA, { GPIO_Pin_12, GPIO_Mode_OUT, GPIO_Speed_10MHz, GPIO_OType_PP, GPIO_PuPd_NOPULL }, PinLow },
[Led2] = { GPIOC, { GPIO_Pin_10, GPIO_Mode_OUT, GPIO_Speed_10MHz, GPIO_OType_PP, GPIO_PuPd_NOPULL }, PinLow },
};
```
https://gcc.gnu.org/onlinedocs/gcc-4.3.2/gcc/Compound-Literals.html

In (ANSI) C99, you can use a designated initializer to initialize a structure:
MY_TYPE a = { .flag = true, .value = 123, .stuff = 0.456 };
Other members are initialized as zero: "Omitted field members are implicitly initialized the same as objects that have static storage duration." (https://gcc.gnu.org/onlinedocs/gcc/Designated-Inits.html)

... abstracts hardware layers from software implementation:

```c
typedef struct port_conf
{
/* for "traditional" MCU pins */
const uint16_t GPIOPin; /*!< GPIO Pin */
const GPIO_TypeDef *GPIOPort; /*!< Port of the Pin */
/* for SIPO pins */
const uint8_t SIPOBit; /*!< Bit location on SIPO */
const uint8_t SIPOPortByte; /*!< Port byte in SIPO buffer */
}port_conf_t;

LOCAL const port_conf_t PinConf[] =
{
 /* SIPO Leds */
 [ LedPort0_pin2 ] = { .SIPOBit = SIPO_BIT_POS_LED_P2, .SIPOPortByte = bsp_ports_Port0 },
 [ LedPort0_pin4 ] = { .SIPOBit = SIPO_BIT_POS_LED_P4, .SIPOPortByte = bsp_ports_Port0 },

 /*SIPO ports */
 [ Power_EN_port0 ] = { .SIPOBit = SIPO_BIT_PORTPOW_EN, .SIPOPortByte = bsp_ports_Port0 },
 [ Power_EN_port1 ] = { .SIPOBit = SIPO_BIT_PORTPOW_EN, .SIPOPortByte = bsp_ports_Port1 },

 /* GPIO Leds */
 [ LedUAL ] = { .GPIOPin = LED_L_Pin, .GPIOPort = LED_L_GPIO_Port },
 [ LedUAR ] = { .GPIOPin = LED_R_Pin, .GPIOPort = LED_R_GPIO_Port },
};
```

Advantages of the solution:
+ All GPIO ports are visible to application layer as same GPIO objects. 
HW differences dosn't obscure application interfaces.
+ Universal interface for all GPIO.
+ Self explanatory, readable and easy port configuration

Disadvantages of the solution:
Code is only compatible with C99 standard and higher (not really a problem as my company software standard )

Conclusion