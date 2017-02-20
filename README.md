#mynewt-nrf52-adc-driver

This package utilizes the mynewt_nordic adc package to add an adc driver for nrf52 bsps.

##usage
Make the upstream repos available locally in your project.yml
```

project.repositories:
    - apache-mynewt-core
    - mynewt_nordic
    - mynewt-nrf52-adc-driver

repository.mynewt-nrf52-adc-driver:
    type: github
    vers: 0-latest
    user: davidgs
    repo: mynewt-nrf52-adc-driver
```

Then add the packge to your projects pkg.yml
```
    - "@mynewt-nrf52-adc-driver/libs/adc_nrf52_driver"
```

Then override the adc settings in your app or target syscfg.yml
```
syscfg.vals:

    ADC_0_SCALING: NRF_ADC_CONFIG_SCALING_SUPPLY_ONE_THIRD
    ADC_0_INPUT: NRF_ADC_CONFIG_INPUT_2
```

Finally call enable the adc and trigger samples presumably as part of a new task

```
/* ADC */
#include <adc/adc.h>
#include "adc_nrf52_driver/adc_nrf52_driver.h"

// /* ADC Task settings */
#define ADC_TASK_PRIO           5
#define ADC_STACK_SIZE          (OS_STACK_ALIGN(32))
struct os_eventq adc_evq;
struct os_task adc_task;
bssnz_t os_stack_t adc_stack[ADC_STACK_SIZE];


int
adc_read_event(struct adc_dev *dev, void *arg, uint8_t etype,
        void *buffer, int buffer_len)
{
    int value;
    int rc;

    value = adc_read(buffer, buffer_len);
    if (value >= 0) {
        console_printf("Got %d\n", value);
    } else {
        console_printf("Error while reading: %d\n", value);
        goto err;
    }
    return (0);
err:
    return (rc);
} 

static void
adc_task_handler(void *unused)
{
    struct adc_dev *adc;
    int rc;
    /* ADC init */
    adc = adc_nrf51_driver_init();
    rc = adc_event_handler_set(adc, adc_read_event, (void *) NULL);
    assert(rc == 0);

    while (1) {
        adc_sample(adc);
        /* Wait 2 second */
        os_time_delay(OS_TICKS_PER_SEC * 2);
    }
}

int
main(void)
{
...
    os_eventq_init(&adc_evq);

    /* Create the ADC reader task.  
     * All sensor operations are performed in this task.
     */
    os_task_init(&adc_task, "sensor", adc_task_handler,
            NULL, ADC_TASK_PRIO, OS_WAIT_FOREVER,
            adc_stack, ADC_STACK_SIZE);
...
```
