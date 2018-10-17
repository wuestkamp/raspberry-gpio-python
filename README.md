What is this?
-----------
This is a "fork" of https://sourceforge.net/projects/raspberry-gpio-python/ which doesn't seem to be maintained any longer.


What is the problem?
-----------
The issue is mentioned here with a solution: https://sourceforge.net/p/raspberry-gpio-python/tickets/111/

The GPIO library on the RaspberryPI has an issue where code like this one

```
while True:
    GPIO.setup(21, GPIO.OUT)
    pwm = GPIO.PWM(21, 50)
    pwm.start(0)

    for i in range(0, 101, 2):
        pwm.ChangeDutyCycle(i)
        time.sleep(0.03)
    for i in range(100, -1, -2):
        pwm.ChangeDutyCycle(i)
        time.sleep(0.03)

    pwm.stop()
    GPIO.cleanup(21)

    time.sleep(1)
```

would stop working after a while without warning. This is because the `pwn.ChangeDutyCycle` creates new pthreads but doesn't clean those up.
This library solves the issue.

Check how many pthreads your PI can have simultaneously
------------------
This is not necessary, just if you want to check your PI. For this you can use the following C code:

```
#include <pthread.h>
#include <stdio.h>

static unsigned long long thread_nr = 0;

pthread_mutex_t mutex_;

void* inc_thread_nr(void* arg) {
    (void*)arg;
    pthread_mutex_lock(&mutex_);
    thread_nr ++;
    pthread_mutex_unlock(&mutex_);

    /* printf("thread_nr = %d\n", thread_nr); */

    sleep(300000);
}

int main(int argc, char *argv[])
{
    int err;
    int cnt = 0;

    pthread_t pid[1000000];

    pthread_mutex_init(&mutex_, NULL);

    while (cnt < 1000000) {

        err = pthread_create(&pid[cnt], NULL, (void*)inc_thread_nr, NULL);
        if (err != 0) {
            break;
        }
        cnt++;
    }

    pthread_join(pid[cnt], NULL);

    pthread_mutex_destroy(&mutex_);
    printf("Maximum number of threads per process is = %d\n", cnt);
}
```

You can compile it via `gcc pthread_test.c -lpthread`. This should print out a number of ~250 or so, depending on your system.
Which means after `ChangeDutyCycle` this function would stop working.

What you have to do to fix this on your PI?
------------------
1. Clone this repo on the raspberry pi
2. cd into the directory
3. run `python setup.py build`
4. Make a backup of the original
`sudo cp /usr/lib/python3/dist-packages/RPi/_GPIO.cpython-35m-arm-linux-gnueabihf.so /usr/lib/python3/dist-packages/RPi/_GPIO.cpython-35m-arm-linux-gnueabihf.so_bak`
5. replace the original
`sudo cp build/lib.linux-armv7l-2.7/RPi/_GPIO.so /usr/lib/python3/dist-packages/RPi/_GPIO.cpython-35m-arm-linux-gnueabihf.so`

Now you should be able to call `ChangeDutyCycle` as often as you like.