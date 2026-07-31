# DESIGNING DAY : 21/7/2026 - 28/7/2026

idea=a 4w mini car that is controllable via phone or tablet,and have some features such as camera and audio

<img width="2121" height="1500" alt="image" src="https://github.com/user-attachments/assets/cf29e163-a043-4861-ace2-ba1ead59ea0e" />
<img width="2121" height="1500" alt="image" src="https://github.com/user-attachments/assets/a197f510-cf32-4c8f-9320-e1d1eef117c9" />

-the screen is a bit hard to make so i use a tiny oled as a prototype first.

# HARWARE BUILD DAY (1): 29/7/2026

-assembled the ESP32 , 4 tt motor module, power (two 18650 battery) , HW-095 module for the motors, OLED screen.

first trying to assemble those raw motors without a breadboard,and i just cut off my jumper wires to get the copper lines.
its quite easy to make,everything just ask google and ai to settle it.
learnt how to connect the battery to esp32 and module for the first time.
i though it will explode
im scared of batteries actually but its okay

after gathering all the materies i found that i dont have a damn base for the whole thing
so i just cut some cardboard and start making everything on top of it
hope i will get a 3d printer soon to print the body
cardboard its just too meh
even popstick is better

after assembling everything,i uploaded the ESP32 code and it initially didnt work.
so why,the website always have bugs damn
but luckily i was able to fix it and tried to control the car
and the damn car really moved
### i am so damn happy when the robot moves man

and so on i keep improving the UI
added joystick,and text to screen,emojo for the car,smile face and other emoji
make it more animated and more human
i like this car
i even named it mini tesla for me hahaha

<img width="2560" height="1920" alt="image" src="https://github.com/user-attachments/assets/842a6b8c-84dd-4512-8d8c-bec483254fc7" />
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/71a2f682-29d4-494b-a7e1-3a430fcef955" />
<img width="1920" height="2560" alt="image" src="https://github.com/user-attachments/assets/e7ef5c05-7511-475c-88d1-65d465c56d8c" />

and this is the photo i took when im making the whole robot,i think it is amazing... 4 hours in midnight!

<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/d3b6ee8c-9356-4230-9abe-23cb30dd6c56" />

# HARDWARE BUILD DAY (2) 31/7/2026

today i added an esp32 CAM to the car
it would have like a view when im controlling a car
doing this is such a burden,required CP2102 module to upload the code
but overall i learnt and its easy to use 
the camera is so lagyyyyyy i dont know why
maybe i need a better camera or something else like a better chip

<img width="1080" height="1920" alt="image" src="https://github.com/user-attachments/assets/a7eeebe8-98d7-4361-bd24-2ab6996c5b44" />
<img width="720" height="1280" alt="image" src="https://github.com/user-attachments/assets/3af5ccc3-960b-4371-944d-018301587dbc" />
<img width="1920" height="2560" alt="image" src="https://github.com/user-attachments/assets/1ac7cf3e-0d41-48e4-9a8d-a271099e145a" />

the UI and the cam page:
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/e6691449-698a-4136-826a-216e3812a7eb" />

quite impressive,but the power draw is crazyyyy, the whole car reboots like every simgle time i push the motor to max limits,i maybe
need a step up or an additional battery to keep it alive.

next upgrade:audio functions.
soon i will try soldering the first time
hope i wont brn my hand









