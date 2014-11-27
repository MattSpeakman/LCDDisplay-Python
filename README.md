LCDDisplay-Python
=================

This file is for the Raspberry Pi. It will take a temperature from a DS18B20 thermistor and output it to a 16x4 LCD display. It will also setup a wireless network if no network connection is available.
#!/usr/bin/python
#
# Fordata LCD Script for
# Raspberry Pi temperature monitoring
#
# Author : Matt Speakman
# Company : KRSS Engineering
# Inspired by a program created by Matt Hawkins
# Date   : 18 April 2013
#

# The wiring for the LCD is as follows:
# 1 : GND
# 2 : 5V
# 3 : Contrast (0-5V)*
# 4 : RS (Register Select)
# 5 : R/W (Read Write)       - GROUND THIS PIN
# 6 : Enable or Strobe
# 7 : Data Bit 0             - NOT USED
# 8 : Data Bit 1             - NOT USED
# 9 : Data Bit 2             - NOT USED
# 10: Data Bit 3             - NOT USED
# 11: Data Bit 4
# 12: Data Bit 5
# 13: Data Bit 6
# 14: Data Bit 7
# 15: LCD Backlight +5V**
# 16: LCD Backlight GND

try:
  import RPi.GPIO as GPIO
  import time
  import smtplib
  import sys
  import os
  import datetime
  import pygame
  from pygame.locals import *
  import urllib2
  GPIO.setwarnings(False)

  #Define the email constants
  fromaddr = 'm.speakman89@googlemail.com'
  toaddrs = get_to_addrs()
  toaddrs = toaddrs[0:-1]
  msg = """From: m.speakman89@googlemail.com
#  To: %s
  Subject: **Temperature Warning**

  A high temperature has been noted""" # % (toaddrs)

  powermessage = """From: m.speakman89@googlemail.com
#  To: %s
  Subject: **Power Failure Warning**

  The power is out""" # % (toaddrs)

  username = 'm.speakman89@googlemail.com'
  password = 'myPassword'

  # Define GPIO to LCD mapping
  LCD_RS = 7
  LCD_E  = 8
  LCD_D4 = 25
  LCD_D5 = 24
  LCD_D6 = 23
  LCD_D7 = 18
  ALARM = 17
  POWER_STATE = 27

  # Define some device constants
  LCD_WIDTH = 16    # Maximum characters per line
  LCD_CHR = True
  LCD_CMD = False

  LCD_LINE_1 = 0x80 # LCD RAM address for the 1st line
  LCD_LINE_2 = 0xC0 # LCD RAM address for the 2nd line
  LCD_LINE_3 = 0x90 # LCD RAM address for the 3rd line
  LCD_LINE_4 = 0xD0 # LCD RAM address for the 4th line

  # Timing constants
  E_PULSE = 0.00005
  E_DELAY = 0.00005

except KeyboardInterrupt:
  print ("Username:")
  user = raw_input()
  print ("Password")
  passwd = raw_input()

  if ("user == pi") and (passwd == "raspberry"):
    #Handle_Interrupt()
    sys.exit()
  else:
    sys.exit()

def WriteInterfaces():
  string="""auto lo

iface lo inet loopback
iface eth0 inet dhcp

allow-hotplug wlan0
iface wlan0 inet manual
wpa-roam /etc/wpa_supplicant/wpa_supplicant.conf
iface default inet dhcp"""

  f = open("/etc/network/interfaces", "w")
  f.write(string)
  f.close()

def InternetOn():
  try:
    response = urllib2.urlopen ('http://www.google.com', timeout = 30)
    return "true"
  except urllib2.URLError as err: pass
  return "false"

def get_to_addrs():
  f = open("email.txt", "r")
  toaddrs = f.read()
  f.close
  return toaddrs

def main():
  try:
    try:
      f = open("email.txt", "r")
      toaddrs = f.read()
      f.close()
    except IOError:
      setup()

    try:
      f = open ("sensor.txt", "r")
      sensor = f.read()
      f.close()
    except IOError:
      sensor_setup()

    checkConnection = InternetOn()  # Comment out this paragraph of commands if you are having problems connecting
    if checkConnection == "true": # To a network and you are stuck in the loop of adding a network
      print "Internet Signal Received" # Everytime you try to start up the program.
    elif checkConnection == "false": #To comment out a line add a hashtag to the start of the line.
      SetupSSID() # The line will change to a red colour to indicate a commented line.

    print"********************"
    print"* KRSS Engineering *"
    print"*  Enviro Monitor  *"
    print"*Program Created by*"
    print"*  Matt Speakman   *"
    print"*mattspeakman.co.uk*"
    print"********************"

    # Main program block
    GPIO.setmode(GPIO.BCM)       # Use BCM GPIO numbers
    GPIO.setup(LCD_E, GPIO.OUT)  # E
    GPIO.setup(LCD_RS, GPIO.OUT) # RS
    GPIO.setup(LCD_D4, GPIO.OUT) # DB4
    GPIO.setup(LCD_D5, GPIO.OUT) # DB5
    GPIO.setup(LCD_D6, GPIO.OUT) # DB6
    GPIO.setup(LCD_D7, GPIO.OUT) # DB7
    GPIO.setup(ALARM, GPIO.OUT) #LED and Buzzer Alarm
    GPIO.setup(POWER_STATE, GPIO.IN) #Power on Alarm

    lcd_init()
    greeting()
    powerInt = 0
    tempInt = 0

    while(True):
      temperature = read_temp()
      fahr = get_fahr(temperature)
      input_bool = get_power_state()

      if (input_bool == 0 and temperature < 60):
        print "No alarm"
        powerInt = 0
        greeting()
        time.sleep(1)
        output(temperature, fahr, input_bool)
        time.sleep(1)

      if (input_bool == 1 and temperature < 60):
        print "Power Alarm"
        power_alarm()
        greeting()
        time.sleep(1)
        output(temperature, fahr, input_bool)
        time.sleep(1)
        if powerInt == 0:
          power_email()
          powerInt = 1

      if (temperature > 60 and input_bool == 0):
        print "Temperature Alarm"
        greeting()
        temp_alarm()
        output(temperature, fahr, input_bool)
        temp_alarm()
        tempInt = tempInt + 1
        if tempInt == 1:
          temp_email()

      if tempInt > 3:
        tempInt = 0

      if (input_bool == 1 and temperature > 60):
        print "Both Alarms"
        greeting()
        both_alarm()
        output(temperature, fahr, input_bool)
        both_alarm()
        tempInt = tempInt + 1
        if powerInt == 0:
          power_email()
          powerInt = 1
        if tempInt == 1:
          temp_email()

  except KeyboardInterrupt:
    print("Username")
    user = raw_input()
    print ("Password")
    passwd = raw_input()

    if (user == "pi") and (passwd == "raspberry"):
     Handle_Interrupt()
     sys.exit()
    else:
      sys.exit()

def Handle_Interrupt():
  path = """# /etc/profile: system-wide .profile file for the Bourne shell (sh(1))
# and Bourne compatible shells (bash(1), ksh(1), ash(1), ...).

if [ "`id -u`" -eq 0 ]; then
  PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
else
  PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/local/games:/usr/games"
fi
export PATH

if [ "$PS1" ]; then
  if [ "$BASH" ] && [ "$BASH" != "/bin/sh" ]; then
    # The file bash.bashrc already sets the default PS1.
    # PS1='\h:\w\$ '
    if [ -f /etc/bash.bashrc ]; then
      . /etc/bash.bashrc
    fi
  else
    if [ "`id -u`" -eq 0 ]; then
      PS1='# '
    else
      PS1='$ '
    fi
  fi
fi

# The default umask is now handled by pam_umask.
# See pam_umask(8) and /etc/login.defs.

if [ -d /etc/profile.d ]; then
  for i in /etc/profile.d/*.sh; do
    if [ -r $i ]; then
      . $i
    fi
  done
  unset i
fi"""
  f = open("/etc/profile", "w")
  f.write(path)
  f.close
  println "File Written"
  print "Now exiting"
  time.sleep(1)


def greeting(): # Standard opening message
  lcd_byte(LCD_LINE_1, LCD_CMD)
  lcd_string("****************")
  lcd_byte(LCD_LINE_2, LCD_CMD)
  lcd_string("KRSS Engineering")
  lcd_byte(LCD_LINE_3, LCD_CMD)
  lcd_string(" Enviro Monitor")
  lcd_byte(LCD_LINE_4, LCD_CMD)
  lcd_string("****************")
  time.sleep(0.5)

def output(temperature, fahr, input_bool): # Print temperature and power state
  try:
    temperature = read_temp()
    fahr = get_fahr(temperature)
    input_bool = get_power_state()
    if input_bool == 0:
      input_string = "Power On"
    elif input_bool == 1:
      input_string = "Power Off"

    lcd_byte(LCD_LINE_1, LCD_CMD)
    lcd_string("Temperature is:")
    lcd_byte(LCD_LINE_2, LCD_CMD)
    lcd_string('%.1f' % (temperature) + " Celsius" )
    lcd_byte(LCD_LINE_3, LCD_CMD )
    lcd_string('%.1f' % (fahr) + " Fahrenheit" )
    lcd_byte(LCD_LINE_4, LCD_CMD)
    lcd_string("" + input_string)
  except Exception:
    lcd_byte(LCD_LINE_1, LCD_CMD)
    lcd_string("Critical Error")
    lcd_byte(LCD_LINE_2, LCD_CMD)
    lcd_string("Can't read temp." )
    lcd_byte(LCD_LINE_3, LCD_CMD )
    lcd_string("Check connection" )
    lcd_byte(LCD_LINE_4, LCD_CMD)
    lcd_string("Then reboot")
    time.sleep(3)
    sys.exit()

def temp_alarm(): # Temperature alarm for LED and buzzer
  GPIO.output(ALARM, True)
  time.sleep(0.5)
  GPIO.output(ALARM, False)
  time.sleep(0.5)

def power_alarm(): # Power off alarm
  GPIO.output(ALARM, True)
  time.sleep(1)
  GPIO.output(ALARM, False)
  time.sleep(1)

def both_alarm():
  GPIO.output(ALARM, True)
  time.sleep(2)
  GPIO.output(ALARM,False)
  time.sleep(1)

def read_temp(): # Read temperature method

  f = open("sensor.txt", "r")
  sensor = f.read()
  f.close()
  sensor = sensor[0:-1]
  string = ('%s' % (sensor) + "/w1_slave")

  try:
#    print"Reading Temperature from file"
    tfile = open("/sys/bus/w1/devices/%s" % (string))
    # Edit the 28-* with the components registered
    text = tfile.read()
    tfile.close()
#    print "Temp read, file closed"
    temperature_data = text.split()[-1]
    temperature = float(temperature_data[2:])
    temperature = temperature / 1000
    return temperature

  except IOError:
    lcd_init()
    display2(LCD_LINE_1, "Read Error")
    time.sleep(2)

def get_fahr(input): # Change celsius to fahrenheit
  try:
    fahr = (input * 1.8) + 32
    return fahr

  except Exception:
    lcd_init()
    display2(LCD_LINE_1, "Error converting")
    display2(LCD_LINE_2, "Celsius to Fahr")
    time.sleep(2)

def log_email(string):
  now = datetime.datetime.now()
  stamp = ("On %s-%s-%s at %s:%s:%s " % (now.day, now.month, now.year, now.hour, now.minute, now.second))
  f = open("log.txt", "a")
  f.write(stamp + string)
  f.write("\n")
  f.close()

def temp_email(): # Send the temperature alert email
  toaddrs = get_to_addrs()
  try:
    server = smtplib.SMTP('smtp.gmail.com')
    server.starttls()
    server.login(username,password)
    server.sendmail(fromaddr, toaddrs, msg)
    server.quit
    log_email("Temperature alert email sent.")

  except Exception as e:
    lcd_init()
    display2(LCD_LINE_1, "Email failed")
    log_email("Email failed. %s" % (e))
    time.sleep(2)

  except EnvironmentError:
    lcd_init()
    display2(LCD_LINE_1, "Network Error")
    log_email("Email failed due to network error")
    time.sleep(2)

def power_email(): # Send the power alert email
  toaddrs = get_to_addrs()
  try:
    server = smtplib.SMTP('smtp.gmail.com')
    server.starttls()
    server.login(username,password)
    server.sendmail(fromaddr, toaddrs, powermessage)
    server.quit
    log_email("Power alert email sent.")
  except Exception as e:
    lcd_init()
    display2(LCD_LINE_1, "Email failed")
    log_email("Power email alert failed. %s" % (e))

def get_power_state(): # Get the power state
  input_value = GPIO.input(POWER_STATE)
  if input_value == False:
    return 0
  elif input_value == True:
    return 1

def display(LINE1, LINE2, LINE3, LINE4):
  lcd_byte(LCD_LINE_1, LCD_CMD)
  lcd_string(LINE1)
  lcd_byte(LCD_LINE_2, LCD_CMD)
  lcd_string(LINE2)
  lcd_byte(LCD_LINE_3, LCD_CMD)
  lcd_string(LINE3)
  lcd_byte(LCD_LINE_4, LCD_CMD)
  lcd_string(LINE4)

def display2(POSITION, STRING): # Sends individual characters with position to the display
  lcd_byte(POSITION, LCD_CMD)
  lcd_string(STRING)

def sensor_setup():
  GPIO.setmode(GPIO.BCM)       # Use BCM GPIO numbers
  GPIO.setup(LCD_E, GPIO.OUT)  # E
  GPIO.setup(LCD_RS, GPIO.OUT) # RS
  GPIO.setup(LCD_D4, GPIO.OUT) # DB4
  GPIO.setup(LCD_D5, GPIO.OUT) # DB5
  GPIO.setup(LCD_D6, GPIO.OUT) # DB6
  GPIO.setup(LCD_D7, GPIO.OUT) # DB7
  GPIO.setup(17, GPIO.OUT) #LED and Buzzer Alarm
  GPIO.setup(27, GPIO.IN) #Power on Alarm
  lcd_init()
  pygame.init()
  screen = pygame.display.set_mode((600, 600))
  pygame.display.set_caption('Pygame Caption')
  pygame.mouse.set_visible(0)
  pygame.event.set_blocked(MOUSEMOTION)
  pygame.event.set_blocked(MOUSEBUTTONUP)
  pygame.event.set_blocked(MOUSEBUTTONDOWN)

  array = []

  myInt= 1
  arrayInt = 0

  display(" ", " ", "Input sensor tag", "Enter to confirm")

  bool = True
  while bool:
    for event in pygame.event.get():
      if (event.key == K_END):
        pygame.quit()
        sys.exit()
      if (event.key == K_BACKSPACE) or (event.key == K_DELETE):
        keypress = " "
        myInt = myInt - 1
        pos = get_pos(myInt)
        display2(pos,keypress)
        num = len(array)
        if num > 0:
          array.pop()

      if myInt >= 33 or myInt <= 0:
        myInt = 1

      if (event.type == pygame.KEYDOWN):
        pos = get_pos(myInt)
        keypress = event.unicode
        display2(pos, keypress)
        display2(LCD_LINE_3, "Input sensor tag")
        display2(LCD_LINE_4, "Enter to confirm")
        array.extend (keypress)
        myInt = myInt + 1
        print event.unicode

        if (event.key == K_TAB) or (event.key == K_CLEAR) or (event.key == K_PAUSE) or (event.key == K_ESCAPE) or  (event.key == K_KP0) or (event.key == K_KP1) or (event.key ==K_KP2) or (event.key ==K_KP3) or (event.key ==K_KP4):
          keypress = " "
          pos = get_pos(myInt)
          display2(pos, keypress)
          myInt = myInt - 1

        if (event.key ==K_KP5) or (event.key ==K_KP6) or (event.key ==K_KP7) or (event.key ==K_KP8) or (event.key ==K_KP9) or (event.key ==K_UP) or (event.key ==K_DOWN) or (event.key ==K_LEFT) or (event.key ==K_RIGHT) or (event.key ==K_INSERT):
          keypress = " "
          pos = get_pos(myInt)
          display2(pos, keypress)
          myInt = myInt - 1

        if (event.key == K_RSHIFT) or (event.key ==K_HOME) or (event.key ==K_PAGEUP) or (event.key ==K_PAGEDOWN) or (event.key ==K_F1) or (event.key ==K_F2) or (event.key ==K_F3) or (event.key ==K_F4) or (event.key ==K_F5):
          keypress = " "
          pos = get_pos(myInt)
          display2(pos, keypress)
          myInt = myInt - 1

        if (event.key ==K_F6) or (event.key ==K_F7) or (event.key ==K_F8) or (event.key ==K_F9) or (event.key ==K_F10) or (event.key ==K_F11) or (event.key ==K_F12) or (event.key ==K_NUMLOCK) or (event.key == K_EURO):
          keypress = " "
          pos = get_pos(myInt)
          display2(pos, keypress)
          myInt = myInt - 1

        if (event.key ==K_LSUPER) or (event.key ==K_RSUPER) or (event.key ==K_MODE) or (event.key ==K_HELP) or (event.key ==K_PRINT) or (event.key ==K_SYSREQ) or (event.key ==K_BREAK) or (event.key ==K_MENU) or (event.key ==K_POWER):
          keypress = " "
          pos = get_pos(myInt)
          display2(pos, keypress)
          myInt = myInt - 1

        if (event.key==K_CAPSLOCK) or (event.key==K_SCROLLOCK) or (event.key==K_NUMLOCK) or (event.key==K_LSHIFT) or (event.key==K_RCTRL) or (event.key==K_LCTRL) or (event.key==K_RALT) or (event.key==K_LALT) or (event.key==K_LMETA) or (event.key==K_RMETA):
          keypress = " "
          pos = get_pos(myInt)
          display2(pos, keypress)
          myInt = myInt - 1

        if (event.key == K_RETURN):
          for item in array:
            f = open ("sensor.txt", "a")
            f.write(item)
            f.close()
            display("Sensor confirmed", "Press END to", "Exit, then reset", "Pi Unit")

def SetupSSID(): # INPUT SSID AND SAVE TO A TEXT FILE
  pygame.init()
  screen = pygame.display.set_mode((600, 600))
  pygame.display.set_caption('Pygame Caption')
  pygame.mouse.set_visible(0)
  pygame.event.set_blocked(MOUSEMOTION)
  pygame.event.set_blocked(MOUSEBUTTUP)
  pygame.event.set_blocked(MOUSEBUTTONDOWN)

  array = []
  GPIO.setmode(GPIO.BCM)       # Use BCM GPIO numbers
  GPIO.setup(LCD_E, GPIO.OUT)  # E
  GPIO.setup(LCD_RS, GPIO.OUT) # RS
  GPIO.setup(LCD_D4, GPIO.OUT) # DB4
  GPIO.setup(LCD_D5, GPIO.OUT) # DB5
  GPIO.setup(LCD_D6, GPIO.OUT) # DB6
  GPIO.setup(LCD_D7, GPIO.OUT) # DB7
  GPIO.setup(17, GPIO.OUT) #LED and Buzzer Alarm
  GPIO.setup(27, GPIO.IN) #Power on Alarm
  lcd_init()

  myInt= 1
  arrayInt = 0

  display(" ", " ", "Input SSID", "Enter to confirm")

  bool = True
  while bool:
    for event in pygame.event.get():
      if (event.key == K_END):
        pygame.quit()
        sys.exit()

      if (event.key == K_BACKSPACE) or (event.key == K_DELETE):
        keypress = " "
        myInt = myInt - 1
        pos = get_pos(myInt)
        display2(pos,keypress)
        display2(LCD_LINE_3, "Input SSID")
        display2(LCD_LINE_4, "Enter to confirm")
        num = len(array)
        if num > 0:
          array.pop()

      if myInt >= 33 or myInt <= 0:
        myInt = 1

      if (event.type == pygame.KEYDOWN):
        pos = get_pos(myInt)
        keypress = event.unicode
        display2(pos, keypress)
        array.extend (keypress)
        myInt = myInt + 1
        print event.unicode
        display2(LCD_LINE_3, "Input SSID")
        display2(LCD_LINE_4, "Enter to confirm")

        if (event.key == K_TAB) or (event.key == K_CLEAR) or (event.key == K_PAUSE) or (event.key == K_ESCAPE) or  (event.key == K_KP0) or (event.key == K_KP1) or (event.key ==K_KP2) or (event.key ==K_KP3) or (event.key ==K_KP4):
          keypress = " "
          pos = get_pos(myInt)
          display2(pos, keypress)
          myInt = myInt - 1

        if (event.key ==K_KP5) or (event.key ==K_KP6) or (event.key ==K_KP7) or (event.key ==K_KP8) or (event.key ==K_KP9) or (event.key ==K_UP) or (event.key ==K_DOWN) or (event.key ==K_LEFT) or (event.key ==K_RIGHT) or (event.key ==K_INSERT):
          keypress = " "
          pos = get_pos(myInt)
          display2(pos, keypress)
          myInt = myInt - 1

        if (event.key ==K_HOME) or (event.key ==K_PAGEUP) or (event.key ==K_PAGEDOWN) or (event.key ==K_F1) or (event.key ==K_F2) or (event.key ==K_F3) or (event.key ==K_F4) or (event.key ==K_F5) or (event.key==K_NUMLOCK):
          keypress = " "
          pos = get_pos(myInt)
          display2(pos, keypress)
          myInt = myInt - 1

        if (event.key ==K_F6) or (event.key ==K_F7) or (event.key ==K_F8) or (event.key ==K_F9) or (event.key ==K_F10) or (event.key ==K_F11) or (event.key ==K_F12) or (event.key ==K_NUMLOCK) or (event.key == K_EURO):
          keypress = " "
          pos = get_pos(myInt)
          display2(pos, keypress)
          myInt = myInt - 1

        if (event.key ==K_LSUPER) or (event.key ==K_RSUPER) or (event.key ==K_MODE) or (event.key ==K_HELP) or (event.key ==K_PRINT) or (event.key ==K_SYSREQ) or (event.key ==K_BREAK) or (event.key ==K_MENU) or (event.key ==K_POWER):
          keypress = " "
          pos = get_pos(myInt)
          display2(pos, keypress)
          myInt = myInt - 1

        if (event.key==K_CAPSLOCK) or (event.key==K_SCROLLOCK) or (event.key==K_RSHIFT) or (event.key==K_LSHIFT) or (event.key==K_RCTRL) or (event.key==K_LCTRL) or (event.key==K_RALT) or (event.key==K_LALT) or (event.key==K_LMETA) or (event.key==K_RMETA):
          keypress = " "
          pos = get_pos(myInt)
          display2(pos, keypress)
          myInt = myInt - 1

        if (event.key == K_RETURN):
          for item in array:
            f = open ("SSID.txt", "a")
            f.write(item)
            f.close() #END SSID SETUP
          SetupPass()

def SetupPass(): # For setting the network Key
  array = []
  myInt= 1
  arrayInt = 0

  display(" ", " ", "Input Password", "Enter to confirm")

  bool = True
  while bool:
    for event in pygame.event.get():
      if (event.key == K_END):
        pygame.quit()
        sys.exit()
      if (event.key == K_BACKSPACE) or (event.key == K_DELETE):
        keypress = " "
        myInt = myInt - 1
        pos = get_pos(myInt)
        display2(pos,keypress)
        num = len(array)
        if num > 0:
          array.pop()

      if myInt >= 33 or myInt <= 0:
        myInt = 1

      if (event.type == pygame.KEYDOWN):
        pos = get_pos(myInt)
        keypress = event.unicode
        display2(pos, keypress)
        display2(LCD_LINE_3, "Input Password")
        display2(LCD_LINE_4, "Enter to confirm")
        array.extend (keypress)
        myInt = myInt + 1
        print event.unicode

        if (event.key == K_TAB) or (event.key == K_CLEAR) or (event.key == K_PAUSE) or (event.key == K_ESCAPE) or  (event.key == K_KP0) or (event.key == K_KP1) or (event.key ==K_KP2) or (event.key ==K_KP3) or (event.key ==K_KP4):
          keypress = " "
          pos = get_pos(myInt)
          display2(pos, keypress)
          myInt = myInt - 1

        if (event.key ==K_KP5) or (event.key ==K_KP6) or (event.key ==K_KP7) or (event.key ==K_KP8) or (event.key ==K_KP9) or (event.key ==K_UP) or (event.key ==K_DOWN) or (event.key ==K_LEFT) or (event.key ==K_RIGHT) or (event.key ==K_INSERT):
          keypress = " "
          pos = get_pos(myInt)
          display2(pos, keypress)
          myInt = myInt - 1

        if (event.key == K_RSHIFT) or (event.key ==K_HOME) or (event.key ==K_PAGEUP) or (event.key ==K_PAGEDOWN) or (event.key ==K_F1) or (event.key ==K_F2) or (event.key ==K_F3) or (event.key ==K_F4) or (event.key ==K_F5):
          keypress = " "
          pos = get_pos(myInt)
          display2(pos, keypress)
          myInt = myInt - 1

        if (event.key ==K_F6) or (event.key ==K_F7) or (event.key ==K_F8) or (event.key ==K_F9) or (event.key ==K_F10) or (event.key ==K_F11) or (event.key ==K_F12) or (event.key ==K_NUMLOCK) or (event.key == K_EURO):
          keypress = " "
          pos = get_pos(myInt)
          display2(pos, keypress)
          myInt = myInt - 1

        if (event.key ==K_LSUPER) or (event.key ==K_RSUPER) or (event.key ==K_MODE) or (event.key ==K_HELP) or (event.key ==K_PRINT) or (event.key ==K_SYSREQ) or (event.key ==K_BREAK) or (event.key ==K_MENU) or (event.key ==K_POWER):
          keypress = " "
          pos = get_pos(myInt)
          display2(pos, keypress)
          myInt = myInt - 1

        if (event.key==K_CAPSLOCK) or (event.key==K_SCROLLOCK) or (event.key==K_RSHIFT) or (event.key==K_LSHIFT) or (event.key==K_RCTRL) or (event.key==K_LCTRL) or (event.key==K_RALT) or (event.key==K_LALT) or (event.key==K_LMETA) or (event.key==K_RMETA):
          keypress = " "
          pos = get_pos(myInt)
          display2(pos, keypress)
          myInt = myInt - 1

        if (event.key == K_RETURN):
          for item in array:
            f = open ("KEY.txt", "a")
            f.write(item)
            f.close()
          SetupProto()

def SetupProto(): # SETUP PROTOCOL
  array = []

  myInt= 1
  arrayInt = 0

  display(" ", " ", "Input RSN or WPA", "WPA2 or WPA1")

  bool = True
  while bool:
    for event in pygame.event.get():
      if (event.key == K_END):
        pygame.quit()
        sys.exit()
      if (event.key == K_BACKSPACE) or (event.key == K_DELETE):
        keypress = " "
        myInt = myInt - 1
        pos = get_pos(myInt)
        display2(pos,keypress)
        num = len(array)
        if num > 0:
          array.pop()

      if myInt >= 33 or myInt <= 0:
        myInt = 1

      if (event.type == pygame.KEYDOWN):
        pos = get_pos(myInt)
        keypress = event.unicode
        display2(pos, keypress)
        display2(LCD_LINE_3, "Input RSN or WPA")
        display2(LCD_LINE_4, "WPA2 or WPA1")
        array.extend (keypress)
        myInt = myInt + 1
        print event.unicode

        if (event.key == K_TAB) or (event.key == K_CLEAR) or (event.key == K_PAUSE) or (event.key == K_ESCAPE) or  (event.key == K_KP0) or (event.key == K_KP1) or (event.key ==K_KP2) or (event.key ==K_KP3) or (event.key ==K_KP4):
          keypress = " "
          pos = get_pos(myInt)
          display2(pos, keypress)
          myInt = myInt - 1

        if (event.key ==K_KP5) or (event.key ==K_KP6) or (event.key ==K_KP7) or (event.key ==K_KP8) or (event.key ==K_KP9) or (event.key ==K_UP) or (event.key ==K_DOWN) or (event.key ==K_LEFT) or (event.key ==K_RIGHT) or (event.key ==K_INSERT):
          keypress = " "
          pos = get_pos(myInt)
          display2(pos, keypress)
          myInt = myInt - 1

        if (event.key == K_RSHIFT) or (event.key ==K_HOME) or (event.key ==K_PAGEUP) or (event.key ==K_PAGEDOWN) or (event.key ==K_F1) or (event.key ==K_F2) or (event.key ==K_F3) or (event.key ==K_F4) or (event.key ==K_F5):
          keypress = " "
          pos = get_pos(myInt)
          display2(pos, keypress)
          myInt = myInt - 1

        if (event.key ==K_F6) or (event.key ==K_F7) or (event.key ==K_F8) or (event.key ==K_F9) or (event.key ==K_F10) or (event.key ==K_F11) or (event.key ==K_F12) or (event.key ==K_NUMLOCK) or (event.key == K_EURO) or (event.key==K_NUMLOCK):
          keypress = " "
          pos = get_pos(myInt)
          display2(pos, keypress)
          myInt = myInt - 1

        if (event.key ==K_LSUPER) or (event.key ==K_RSUPER) or (event.key ==K_MODE) or (event.key ==K_HELP) or (event.key ==K_PRINT) or (event.key ==K_SYSREQ) or (event.key ==K_BREAK) or (event.key ==K_MENU) or (event.key ==K_POWER):
          keypress = " "
          pos = get_pos(myInt)
          display2(pos, keypress)
          myInt = myInt - 1

        if (event.key==K_CAPSLOCK) or (event.key==K_SCROLLOCK) or (event.key==K_RSHIFT) or (event.key==K_LSHIFT) or (event.key==K_RCTRL) or (event.key==K_LCTRL) or (event.key==K_RALT) or (event.key==K_LALT) or (event.key==K_LMETA) or (event.key==K_RMETA):
          keypress = " "
          pos = get_pos(myInt)
          display2(pos, keypress)
          myInt = myInt - 1

        if (event.key == K_RETURN):
          for item in array:
            f = open ("PROTO.txt", "a")
            f.write(item)
            f.close()
          SetupKeyMgmt() #END PROTOCOL SETUP

def SetupKeyMgmt(): #DEFINE WPA-PSK or WPA-EAP
  array = []

  myInt= 1
  arrayInt = 0

  display(" ", " ", "WPA-PSK - Shared", "WPA-EAP - Enterp")

  bool = True
  while bool:
    for event in pygame.event.get():
      if (event.key == K_END):
        pygame.quit()
        sys.exit()
      if (event.key == K_BACKSPACE) or (event.key == K_DELETE):
        keypress = " "
        myInt = myInt - 1
        pos = get_pos(myInt)
        display2(pos,keypress)
        num = len(array)
        if num > 0:
          array.pop()

      if myInt >= 33 or myInt <= 0:
        myInt = 1

      if (event.type == pygame.KEYDOWN):
        pos = get_pos(myInt)
        keypress = event.unicode
        display2(pos, keypress)
        display2(LCD_LINE_3, "WPA-PSK - Shared")
        display2(LCD_LINE_4, "WPA-EAP - Enterp")
        array.extend (keypress)
        myInt = myInt + 1
        print event.unicode

        if (event.key == K_TAB) or (event.key == K_CLEAR) or (event.key == K_PAUSE) or (event.key == K_ESCAPE) or  (event.key == K_KP0) or (event.key == K_KP1) or (event.key ==K_KP2) or (event.key ==K_KP3) or (event.key ==K_KP4):
          keypress = " "
          pos = get_pos(myInt)
          display2(pos, keypress)
          myInt = myInt - 1

        if (event.key ==K_KP5) or (event.key ==K_KP6) or (event.key ==K_KP7) or (event.key ==K_KP8) or (event.key ==K_KP9) or (event.key ==K_UP) or (event.key ==K_DOWN) or (event.key ==K_LEFT) or (event.key ==K_RIGHT) or (event.key ==K_INSERT):
          keypress = " "
          pos = get_pos(myInt)
          display2(pos, keypress)
          myInt = myInt - 1

        if (event.key == K_RSHIFT) or (event.key ==K_HOME) or (event.key ==K_PAGEUP) or (event.key ==K_PAGEDOWN) or (event.key ==K_F1) or (event.key ==K_F2) or (event.key ==K_F3) or (event.key ==K_F4) or (event.key ==K_F5):
          keypress = " "
          pos = get_pos(myInt)
          display2(pos, keypress)
          myInt = myInt - 1

        if (event.key ==K_F6) or (event.key ==K_F7) or (event.key ==K_F8) or (event.key ==K_F9) or (event.key ==K_F10) or (event.key ==K_F11) or (event.key ==K_F12) or (event.key ==K_NUMLOCK) or (event.key == K_EURO) or (event.key==K_NUMLOCK):
          keypress = " "
          pos = get_pos(myInt)
          display2(pos, keypress)
          myInt = myInt - 1

        if (event.key ==K_LSUPER) or (event.key ==K_RSUPER) or (event.key ==K_MODE) or (event.key ==K_HELP) or (event.key ==K_PRINT) or (event.key ==K_SYSREQ) or (event.key ==K_BREAK) or (event.key ==K_MENU) or (event.key ==K_POWER):
          keypress = " "
          pos = get_pos(myInt)
          display2(pos, keypress)
          myInt = myInt - 1

        if (event.key==K_CAPSLOCK) or (event.key==K_SCROLLOCK) or (event.key==K_RSHIFT) or (event.key==K_LSHIFT) or (event.key==K_RCTRL) or (event.key==K_LCTRL) or (event.key==K_RALT) or (event.key==K_LALT) or (event.key==K_LMETA) or (event.key==K_RMETA):
          keypress = " "
          pos = get_pos(myInt)
          display2(pos, keypress)
          myInt = myInt - 1

        if (event.key == K_RETURN):
          for item in array:
            f = open ("KEYMGMT.txt", "a")
            f.write(item)
            f.close()
          SetupPairWise()

def SetupPairWise(): # Define TKIP or CMMP
  array = []

  myInt= 1
  arrayInt = 0

  display(" ", " ", "TKIP for WPA2/1", "or CMMP")

  bool = True
  while bool:
    for event in pygame.event.get():
      if (event.key == K_END):
        pygame.quit()
        sys.exit()
      if (event.key == K_BACKSPACE) or (event.key == K_DELETE):
        keypress = " "
        myInt = myInt - 1
        pos = get_pos(myInt)
        display2(pos,keypress)
        num = len(array)
        if num > 0:
          array.pop()

      if myInt >= 33 or myInt <= 0:
        myInt = 1

      if (event.type == pygame.KEYDOWN):
        pos = get_pos(myInt)
        keypress = event.unicode
        display2(pos, keypress)
        display2(LCD_LINE_3, "TKIP for WPA2/1")
        display2(LCD_LINE_4, "or CMMP")
        array.extend (keypress)
        myInt = myInt + 1
        print event.unicode

        if (event.key == K_TAB) or (event.key == K_CLEAR) or (event.key == K_PAUSE) or (event.key == K_ESCAPE) or  (event.key == K_KP0) or (event.key == K_KP1) or (event.key ==K_KP2) or (event.key ==K_KP3) or (event.key ==K_KP4):
          keypress = " "
          pos = get_pos(myInt)
          display2(pos, keypress)
          myInt = myInt - 1

        if (event.key ==K_KP5) or (event.key ==K_KP6) or (event.key ==K_KP7) or (event.key ==K_KP8) or (event.key ==K_KP9) or (event.key ==K_UP) or (event.key ==K_DOWN) or (event.key ==K_LEFT) or (event.key ==K_RIGHT) or (event.key ==K_INSERT):
          keypress = " "
          pos = get_pos(myInt)
          display2(pos, keypress)
          myInt = myInt - 1

        if (event.key == K_RSHIFT) or (event.key ==K_HOME) or (event.key ==K_PAGEUP) or (event.key ==K_PAGEDOWN) or (event.key ==K_F1) or (event.key ==K_F2) or (event.key ==K_F3) or (event.key ==K_F4) or (event.key ==K_F5):
          keypress = " "
          pos = get_pos(myInt)
          display2(pos, keypress)
          myInt = myInt - 1

        if (event.key ==K_F6) or (event.key ==K_F7) or (event.key ==K_F8) or (event.key ==K_F9) or (event.key ==K_F10) or (event.key ==K_F11) or (event.key ==K_F12) or (event.key ==K_NUMLOCK) or (event.key == K_EURO) or (event.key==K_NUMLOCK):
          keypress = " "
          pos = get_pos(myInt)
          display2(pos, keypress)
          myInt = myInt - 1

        if (event.key ==K_LSUPER) or (event.key ==K_RSUPER) or (event.key ==K_MODE) or (event.key ==K_HELP) or (event.key ==K_PRINT) or (event.key ==K_SYSREQ) or (event.key ==K_BREAK) or (event.key ==K_MENU) or (event.key ==K_POWER):
          keypress = " "
          pos = get_pos(myInt)
          display2(pos, keypress)
          myInt = myInt - 1

        if (event.key==K_CAPSLOCK) or (event.key==K_SCROLLOCK) or (event.key==K_RSHIFT) or (event.key==K_LSHIFT) or (event.key==K_RCTRL) or (event.key==K_LCTRL) or (event.key==K_RALT) or (event.key==K_LALT) or (event.key==K_LMETA) or (event.key==K_RMETA):
          keypress = " "
          pos = get_pos(myInt)
          display2(pos, keypress)
          myInt = myInt - 1

        if (event.key == K_RETURN):
          for item in array:
            f = open ("PAIRWISE.txt", "a")
            f.write(item)
            f.close()
          SetupAuth() # END PAIRWISE DECLARATION

def SetupAuth(): # OPEN/SHARED/LEAP
  array = []

  myInt= 1
  arrayInt = 0

  display(" ", " ", "OPEN/SHARED/LEAP", "Normally OPEN")

  bool = True
  while bool:
    for event in pygame.event.get():
      if (event.key == K_END):
        pygame.quit()
        sys.exit()
      if (event.key == K_BACKSPACE) or (event.key == K_DELETE):
        keypress = " "
        myInt = myInt - 1
        pos = get_pos(myInt)
        display2(pos,keypress)
        num = len(array)
        if num > 0:
          array.pop()

      if myInt >= 33 or myInt <= 0:
        myInt = 1

      if (event.type == pygame.KEYDOWN):
        pos = get_pos(myInt)
        keypress = event.unicode
        display2(pos, keypress)
        display2(LCD_LINE_3, "OPEN/SHARED/LEAP")
        display2(LCD_LINE_4, "Normally OPEN")
        array.extend (keypress)
        myInt = myInt + 1
        print event.unicode

        if (event.key == K_TAB) or (event.key == K_CLEAR) or (event.key == K_PAUSE) or (event.key == K_ESCAPE) or  (event.key == K_KP0) or (event.key == K_KP1) or (event.key ==K_KP2) or (event.key ==K_KP3) or (event.key ==K_KP4):
          keypress = " "
          pos = get_pos(myInt)
          display2(pos, keypress)
          myInt = myInt - 1

        if (event.key ==K_KP5) or (event.key ==K_KP6) or (event.key ==K_KP7) or (event.key ==K_KP8) or (event.key ==K_KP9) or (event.key ==K_UP) or (event.key ==K_DOWN) or (event.key ==K_LEFT) or (event.key ==K_RIGHT) or (event.key ==K_INSERT):
          keypress = " "
          pos = get_pos(myInt)
          display2(pos, keypress)
          myInt = myInt - 1

        if (event.key == K_RSHIFT) or (event.key ==K_HOME) or (event.key ==K_PAGEUP) or (event.key ==K_PAGEDOWN) or (event.key ==K_F1) or (event.key ==K_F2) or (event.key ==K_F3) or (event.key ==K_F4) or (event.key ==K_F5):
          keypress = " "
          pos = get_pos(myInt)
          display2(pos, keypress)
          myInt = myInt - 1

        if (event.key ==K_F6) or (event.key ==K_F7) or (event.key ==K_F8) or (event.key ==K_F9) or (event.key ==K_F10) or (event.key ==K_F11) or (event.key ==K_F12) or (event.key ==K_NUMLOCK) or (event.key == K_EURO) or (event.key==K_NUMLOCK):
          keypress = " "
          pos = get_pos(myInt)
          display2(pos, keypress)
          myInt = myInt - 1

        if (event.key ==K_LSUPER) or (event.key ==K_RSUPER) or (event.key ==K_MODE) or (event.key ==K_HELP) or (event.key ==K_PRINT) or (event.key ==K_SYSREQ) or (event.key ==K_BREAK) or (event.key ==K_MENU) or (event.key ==K_POWER):
          keypress = " "
          pos = get_pos(myInt)
          display2(pos, keypress)
          myInt = myInt - 1

        if (event.key==K_CAPSLOCK) or (event.key==K_SCROLLOCK) or (event.key==K_RSHIFT) or (event.key==K_LSHIFT) or (event.key==K_RCTRL) or (event.key==K_LCTRL) or (event.key==K_RALT) or (event.key==K_LALT) or (event.key==K_LMETA) or (event.key==K_RMETA):
          keypress = " "
          pos = get_pos(myInt)
          display2(pos, keypress)
          myInt = myInt - 1

        if (event.key == K_RETURN):
          for item in array:
            f = open ("AUTH.txt", "a")
            f.write(item)
            f.close()
          WriteNetwork()

def WriteNetwork():
  f = open("SSID.txt", "r")
  obj1 = f.read()
  str1 = obj1[0:-1]
  f.close()
  f = open("KEY.txt", "r")
  obj2 = f.read()
  str2 = obj2[0:-1]
  f.close()
  f = open("PROTO.txt", "r")
  obj3 = f.read()
  str3 = obj3[0:-1]
  f.close()
  f= open("KEYMGMT.txt", "r")
  obj4 = f.read()
  str4 = obj4[0:-1]
  f.close()
  f = open("PAIRWISE.txt", "r")
  obj5 = f.read()
  str5 = obj5[0:-1]
  f.close()
  f = open("AUTH.txt", "r")
  obj6 = f.read()
  str6 = obj6[0:-1]
  f.close()

  string = """network={
        ssid="%s"
        psk="%s"
        proto=%s #Protocol can be (RSN for WP2) and (WPA for WPA1)
        key_mgmt=%s # Key management can be (WPA-PSK for Pre-Shared) or (WPA-EAP for Enterprise)
        pairwise=%s # Pairwise can be (TKIP for WPA2 or WPA1) or CMMP
        auth_alg=%s # (Open for WPA1/2) Less commonly used are SHARED or LEAP
}""" % (str1, str2, str3, str4, str5, str6)

  f = open("/etc/wpa_supplicant/wpa_supplicant.conf", "a")
  f.write(string)
  f.close()

  display("Network Setup", "Complete", "Raspberry Pi", "Will now reboot")

  os.popen2("sudo ifup --force wlan0")
  os.popen2("sudo rm SSID.txt KEY.txt PROTO.txt KEYMGMT.txt PAIRWISE.txt AUTH.txt")
  WriteInterfaces()
  os.popen2("sudo shutdown -r now")

def setup(): # For running the setup program
  GPIO.setmode(GPIO.BCM)       # Use BCM GPIO numbers
  GPIO.setup(LCD_E, GPIO.OUT)  # E
  GPIO.setup(LCD_RS, GPIO.OUT) # RS
  GPIO.setup(LCD_D4, GPIO.OUT) # DB4
  GPIO.setup(LCD_D5, GPIO.OUT) # DB5
  GPIO.setup(LCD_D6, GPIO.OUT) # DB6
  GPIO.setup(LCD_D7, GPIO.OUT) # DB7
  GPIO.setup(17, GPIO.OUT) #LED and Buzzer Alarm
  GPIO.setup(27, GPIO.IN) #Power on Alarm

  lcd_init()
  pygame.init()
  screen = pygame.display.set_mode((600, 600))
  pygame.display.set_caption('Pygame Caption')
  pygame.mouse.set_visible(0)
  pygame.event.set_blocked(MOUSEMOTION)
  pygame.event.set_blocked(MOUSEBUTTONDOWN)
  pygame.event.set_blocked(MOUSEBUTTONUP)

  array = []

  myInt= 1
  arrayInt = 0

  display(" ", " ", "Input email", "Enter to confirm")

  bool = True
  while bool:
    for event in pygame.event.get():
      if (event.key == K_END):
        pygame.quit()
        sys.exit()
      if (event.key == K_BACKSPACE) or (event.key == K_DELETE):
        keypress = " "
        myInt = myInt - 1
        pos = get_pos(myInt)
        display2(pos,keypress)
        num = len(array)
        if num > 0:
          array.pop()

      if myInt >= 33 or myInt <= 0:
        myInt = 1

      if (event.type == pygame.KEYDOWN):
        pos = get_pos(myInt)
        keypress = event.unicode
        display2(pos, keypress)
        display2(LCD_LINE_3, "Input email")
        display2(LCD_LINE_4, "Enter to confirm")
        array.extend (keypress)
        myInt = myInt + 1
        print event.unicode

        if (event.key == K_TAB) or (event.key == K_CLEAR) or (event.key == K_PAUSE) or (event.key == K_ESCAPE) or  (event.key == K_KP0) or (event.key == K_KP1) or (event.key ==K_KP2) or (event.key ==K_KP3) or (event.key ==K_KP4):
          keypress = " "
          pos = get_pos(myInt)
          display2(pos, keypress)
          myInt = myInt - 1

        if (event.key ==K_KP5) or (event.key ==K_KP6) or (event.key ==K_KP7) or (event.key ==K_KP8) or (event.key ==K_KP9) or (event.key ==K_UP) or (event.key ==K_DOWN) or (event.key ==K_LEFT) or (event.key ==K_RIGHT) or (event.key ==K_INSERT):
          keypress = " "
          pos = get_pos(myInt)
          display2(pos, keypress)
          myInt = myInt - 1

        if (event.key == K_RSHIFT) or (event.key ==K_HOME) or (event.key ==K_PAGEUP) or (event.key ==K_PAGEDOWN) or (event.key ==K_F1) or (event.key ==K_F2) or (event.key ==K_F3) or (event.key ==K_F4) or (event.key ==K_F5):
          keypress = " "
          pos = get_pos(myInt)
          display2(pos, keypress)
          myInt = myInt - 1

        if (event.key ==K_F6) or (event.key ==K_F7) or (event.key ==K_F8) or (event.key ==K_F9) or (event.key ==K_F10) or (event.key ==K_F11) or (event.key ==K_F12) or (event.key ==K_NUMLOCK) or (event.key == K_EURO)or (event.key==K_NUMLOCK):
          keypress = " "
          pos = get_pos(myInt)
          display2(pos, keypress)
          myInt = myInt - 1

        if (event.key==K_CAPSLOCK) or (event.key==K_SCROLLOCK) or (event.key==K_RSHIFT) or (event.key==K_LSHIFT) or (event.key==K_RCTRL) or (event.key==K_LCTRL) or (event.key==K_RALT) or (event.key==K_LALT) or (event.key==K_LMETA) or (event.key==K_RMETA):
          keypress = " "
          pos = get_pos(myInt)
          display2(pos, keypress)
          myInt = myInt - 1

        if (event.key ==K_LSUPER) or (event.key ==K_RSUPER) or (event.key ==K_MODE) or (event.key ==K_HELP) or (event.key ==K_PRINT) or (event.key ==K_SYSREQ) or (event.key ==K_BREAK) or (event.key ==K_MENU) or (event.key ==K_POWER):
          keypress = " "
          pos = get_pos(myInt)
          display2(pos, keypress)
          myInt = myInt - 1

        if (event.key == K_RETURN):
          for item in array:
            f = open ("email.txt", "a")
            f.write(item)
            f.close()
          display("Email confirmed", "Press END to", "Exit, then reset", "Pi Unit")

def get_pos(i):
  if i <= 0:
    return 0x80
  if i == 1:
    return 0x80
  if i == 2:
    return 0x81
  if i == 3:
    return 0x82
  if i == 4:
    return 0x83
  if i == 5:
    return 0x84
  if i == 6:
    return 0x85
  if i == 7:
    return 0x86
  if i == 8:
    return 0x87
  if i == 9:
    return 0x88
  if i == 10:
    return 0x89
  if i == 11:
    return 0x8A
  if i == 12:
    return 0x8B
  if i == 13:
    return 0x8C
  if i == 14:
    return 0x8D
  if i == 15:
    return 0x8E
  if i == 16:
    return 0x8F

  if i == 17:
    return 0xC0
  if i == 18:
    return 0xC1
  if i == 19:
    return 0xC2
  if i == 20:
    return 0xC3
  if i == 21:
    return 0xC4
  if i == 22:
    return 0xC5
  if i == 23:
    return 0xC6
  if i == 24:
    return 0xC7
  if i == 25:
    return 0xC8
  if i == 26:
    return 0xC9
  if i == 27:
    return 0xCA
  if i == 28:
    return 0xCB
  if i == 29:
    return 0xCC
  if i == 30:
    return 0xCD
  if i == 31:
    return 0xCE
  if i == 32:
    return 0xCF

def lcd_init():  # Initialise display
  lcd_byte(0x33,LCD_CMD)
  lcd_byte(0x32,LCD_CMD)
  lcd_byte(0x28,LCD_CMD)
  lcd_byte(0x0C,LCD_CMD)
  lcd_byte(0x06,LCD_CMD)
  lcd_byte(0x01,LCD_CMD)

def lcd_string(message):  # Send string to display
  message = message.ljust(LCD_WIDTH," ")
  for i in range(LCD_WIDTH):
    lcd_byte(ord(message[i]),LCD_CHR)

def lcd_byte(bits, mode):
  # Send byte to data pins
  # bits = data
  # mode = True  for character
  #        False for command

  GPIO.output(LCD_RS, mode) # RS
  # High bits
  GPIO.output(LCD_D4, False)
  GPIO.output(LCD_D5, False)
  GPIO.output(LCD_D6, False)
  GPIO.output(LCD_D7, False)
  if bits&0x10==0x10:
    GPIO.output(LCD_D4, True)
  if bits&0x20==0x20:
    GPIO.output(LCD_D5, True)
  if bits&0x40==0x40:
    GPIO.output(LCD_D6, True)
  if bits&0x80==0x80:
    GPIO.output(LCD_D7, True)

  # Toggle 'Enable' pin
  time.sleep(E_DELAY)
  GPIO.output(LCD_E, True)
  time.sleep(E_PULSE)
  GPIO.output(LCD_E, False)
  time.sleep(E_DELAY)

  # Low bits
  GPIO.output(LCD_D4, False)
  GPIO.output(LCD_D5, False)
  GPIO.output(LCD_D6, False)
  GPIO.output(LCD_D7, False)
  if bits&0x01==0x01:
    GPIO.output(LCD_D4, True)
  if bits&0x02==0x02:
    GPIO.output(LCD_D5, True)
  if bits&0x04==0x04:
    GPIO.output(LCD_D6, True)
  if bits&0x08==0x08:
    GPIO.output(LCD_D7, True)
  # Toggle 'Enable' pin
  time.sleep(E_DELAY)
  GPIO.output(LCD_E, True)
  time.sleep(E_PULSE)
  GPIO.output(LCD_E, False)
  time.sleep(E_DELAY)

if __name__ == '__main__':
  main()

