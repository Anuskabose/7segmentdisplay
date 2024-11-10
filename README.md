# 7segmentdisplay
#include <stdio.h>
#include "pico/stdlib.h"
#include "hardware/gpio.h"
#include "hardware/adc.h"
#include "hardware/pwm.h"

#define FIRST_GPIO 2
#define BUTTON_GPIO (FIRST_GPIO+7)
#define LED_GPIO 14
#define LED_GND_GPIO 15  // Added to define the other GPIO pin for LED
#define BUZZER_GPIO 17
#define POTENTIOMETER_ADC 26  // ADC channel for potentiometer (GPIO 26)

// This array converts a number 0-9 to a bit pattern to send to the GPIOs
int bits[10] = {
        0x3f,  // 0
        0x06,  // 1
        0x5b,  // 2
        0x4f,  // 3
        0x66,  // 4
        0x6d,  // 5
        0x7d,  // 6
        0x07,  // 7
        0x7f,  // 8
        0x67   // 9
};

// Initialize ADC for potentiometer
void init_adc() {
    adc_init();
    adc_gpio_init(POTENTIOMETER_ADC);  // Initialize GPIO 26 for potentiometer (ADC)
    adc_select_input(0);  // Select ADC input 0 (GPIO 26)
}

// Read the value from the potentiometer
uint16_t read_potentiometer() {
    return adc_read();  // Returns a 12-bit value (0 to 4095)
}

// Initialize PWM for a GPIO pin
void init_pwm(uint gpio) {
    gpio_set_function(gpio, GPIO_FUNC_PWM);  // Set GPIO for PWM
    uint slice_num = pwm_gpio_to_slice_num(gpio);  // Get PWM slice number
    pwm_set_wrap(slice_num, 255);  // Set PWM period to 255 for 8-bit control
    pwm_set_chan_level(slice_num, pwm_gpio_to_channel(gpio), 0);  // Initial value 0
    pwm_set_enabled(slice_num, true);  // Enable PWM
}

// Set the PWM duty cycle for the LED
void set_pwm_duty(uint gpio, uint duty) {
    uint slice_num = pwm_gpio_to_slice_num(gpio);
    pwm_set_chan_level(slice_num, pwm_gpio_to_channel(gpio), duty);
}

int main() {
    stdio_init_all();
    printf("Hello, 7segment with LED Brightness Control via Potentiometer!\n");

    // Initialize the 7-segment display GPIOs
    for (int gpio = FIRST_GPIO; gpio < FIRST_GPIO + 7; gpio++) {
        gpio_init(gpio);
        gpio_set_dir(gpio, GPIO_OUT);
        gpio_set_outover(gpio, GPIO_OVERRIDE_INVERT);  // Invert to pull low for LED on
    }

    // Initialize button
    gpio_init(BUTTON_GPIO);
    gpio_set_dir(BUTTON_GPIO, GPIO_IN);
    gpio_pull_up(BUTTON_GPIO);

    // Initialize LED
    gpio_init(LED_GPIO);
    gpio_set_dir(LED_GPIO, GPIO_OUT);
    gpio_init(LED_GND_GPIO);   // Initialize the GND pin for the LED
    gpio_set_dir(LED_GND_GPIO, GPIO_OUT);
    gpio_put(LED_GND_GPIO, 0); // Set GND pin for LED to low (ground)
    init_pwm(LED_GPIO);        // Initialize PWM for LED

    // Initialize buzzer
    gpio_init(BUZZER_GPIO);
    gpio_set_dir(BUZZER_GPIO, GPIO_OUT);

    // Initialize ADC for potentiometer
    init_adc();

    int val = 0;
    bool buzzer_on = false;

    while (true) {
        // Check the button press to increment or decrement the counter
        if (!gpio_get(BUTTON_GPIO)) {
            if (val == 9) {
                val = 0;
            } else {
                val++;
            }
        }

        // Buzzer should buzz when the counter reaches 0
        if (val == 0) {
            gpio_put(BUZZER_GPIO, 1);  // Turn on buzzer
            buzzer_on = true;
        } else {
            if (buzzer_on) {
                gpio_put(BUZZER_GPIO, 0);  // Turn off buzzer
                buzzer_on = false;
            }
        }

        // Read the potentiometer value and control LED brightness
        uint16_t pot_value = read_potentiometer();  // Read potentiometer (0 to 4095)
        uint16_t pwm_value = pot_value >> 4;        // Scale it down to 8-bit (0 to 255)

        // Control LED brightness using PWM
        set_pwm_duty(LED_GPIO, pwm_value);

        // Set the 7-segment display
        int32_t mask = bits[val] << FIRST_GPIO;
        gpio_set_mask(mask);
        sleep_ms(250);  // Add a delay to avoid too frequent updates
        gpio_clr_mask(mask);
    }
}
