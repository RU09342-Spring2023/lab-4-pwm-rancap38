# Software PWM
Most microprocessors will have a Timer module, but depending on the device, some may not come with pre-built PWM modules. Instead, you may have to utilize software techniques to synthesize PWM on your own.

## Task
You need to generate a 1kHz PWM signal with a duty cycle between 0% and 100%. Upon the processor starting up, you should PWM both of the on-board LEDs at a 50% duty cycle. Upon pressing the on-board buttons, the duty cycle of the LEDs should increase by 10%, based on which button you press. Once you have reached 100%, your duty cycle should go back to 0% on the next button press.
 - Button 2.1 Should control LED 1.0
 - Button 4.3 should control LED 6.6

## Deliverables
You will need to upload the .c file and a README explaining your code and any design decisions made.

### Hints
You really, really, really, really need to hook up the output of your LED pin to an oscilloscope to make sure that the duty cycle is accurate. Also, since you are going to be doing a lot of initialization, it would be helpful for all persons involved if you created your main function like:

```c
int main(void)
{
	WDTCTL = WDTPW | WDTHOLD;	// stop watchdog timer
	LEDSetup(); // Initialize our LEDS
	ButtonSetup();  // Initialize our button
	TimerA0Setup(); // Initialize Timer0
	TimerA1Setup(); // Initialize Timer1
	__bis_SR_register(LPM0_bits + GIE);       // Enter LPM0 w/ interrupt
}
```

This way, each of the steps in initialization can be isolated for easier understanding and debugging.


## Extra Work
### Linear Brightness
Much like every other things with humans, not everything we interact with we perceive as linear. For senses such as sight or hearing, certain features such as volume or brightness have a logarithmic relationship with our senses. Instead of just incrementing by 10%, try making the brightness appear to change linearly.

Lab Code for part 1c: 
/*
 *
 *
 *  Created on: 5/6/2023
 *  Author: Derek Rancapan
 *
 *
 */

#include <msp430.h>

// Initialize the LEDs
void gpioInit() {

  P1DIR |= BIT0;    // Pin 1.0 Output
  P1OUT &= ~BIT0;   // Clear pin
  P6DIR |= BIT6;    // Pin 6.6 Output
  P6OUT &= ~BIT6;   // Clear pin


  P4DIR &= ~BIT1;   // Configure P4.1 input
  P4OUT |= BIT1;
  P4REN |= BIT1;    // Pull up resistor
  P4IE  |= BIT1;    // Interrupt
  P4IES &= ~BIT1;


  P2DIR &= ~BIT3;   // Configure P2.3 Input
  P2OUT |= BIT3;
  P2REN |= BIT3;    // pull up
  P2IES &= ~BIT3;   // interrupt
  P2IE |= BIT3;
}

// Setup the timers for PWM
void timerInit() {

  // Initialize Timer 0
  TB0CTL = TBSSEL__SMCLK | MC__UP | TBIE;   // Enable: TimerB0 SMCLK, Up-Mode, Interrupt
  TB0CCTL1 |= CCIE; // Capture/compare interrupt enable
  TB0CCR0 = 1000;   // f = 1kHz
  TB0CCR1 = 500;    // 50 percent duty cycle

  // Initialize Timer 1
  TB1CTL = TBSSEL__SMCLK | MC__UP | TBIE;   // Enable: TimerB1 SMCLK  Up-Mode  Interrupt Enabled
  TB1CCTL1 |= CCIE; // Capture/compare interrupt enable
  TB1CCR0 = 1000;   // 1kHz
  TB1CCR1 = 500;    // Sets the duty cycle to 50%

}

void main() {

  WDTCTL = WDTPW | WDTHOLD; // Disable watchdog timer

  gpioInit();   // Initialize pins

  timerInit(); // Setup the timers for PWM

  PM5CTL0 &= ~LOCKLPM5;

  __bis_SR_register(LPM0_bits + GIE);   // Low power mode
  __no_operation();
}

// Button 2.3 interrupt (Red LED)
#pragma vector=PORT2_VECTOR
__interrupt void P2_ISR() {

  P2IFG &= ~BIT3;   // Clear interrupt flag

  if (TB0CCR1 >= 1000)   // Check if at 100% duty cycle
    TB0CCR1 = 1;    // Set duty cycle to 0%

  else
    TB0CCR1 += 100; // Increase duty cycle by 10%

}

// Button 4.1 interrupt (Green LED)
#pragma vector=PORT4_VECTOR
__interrupt void P4_ISR() {

  P4IFG &= ~BIT1;   // Clear interrupt flag

  if (TB1CCR1 >= 1000)  // Check if at 100% duty cycle
    TB1CCR1 = 1;    // Set duty cycle to 0%
  else
    TB1CCR1 += 100; // Increase duty cycle by 10%

}

// PWM 0 interrupt
#pragma vector=TIMER0_B1_VECTOR
__interrupt void T0_ISR() {

  switch (__even_in_range(TB0IV, TB0IV_TBIFG)) {    // Check if rising or falling edge

    case TB0IV_NONE:
      break;

    case TB0IV_TBCCR1:
      P1OUT &= ~BIT0;   // Turn LED off if falling edge
      break;

    case TB0IV_TBCCR2:
      break;

    case TB0IV_TBIFG:
      P1OUT |= BIT0;    // Turn LED on if rising edge
      break;

    default:
      break;

  }

}

// PWM 1 interrupt
#pragma vector=TIMER1_B1_VECTOR
__interrupt void T1_ISR() {

  switch (__even_in_range(TB1IV, TB1IV_TBIFG)) {    // Check if rising or falling edge

    case TB1IV_NONE:
      break;

    case TB1IV_TBCCR1:
      P6OUT &= ~BIT6;   // Turn LED off if falling edge
      break;

    case TB1IV_TBCCR2:
      break;

    case TB1IV_TBIFG:
      P6OUT |= BIT6;    // Turn LED on if rising edge
      break;

    default:
      break;

  }
}
