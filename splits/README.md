# Splits - Interval Training Software
Repo: https://github.com/kate-holdener/splits

The software integrates with NFC reader hardware (ACR122U) and RFID scanner to monitor athletes' interval training, as they are running on the track. The NFC reader is used to detect the start time - an athlete scans their NFC tag before running off. The software keeps track of each athlete's interval times and rest times. The UI has a list of resting athletes, and when an athlete's rest time is up, the UI tells the athlete to run again.

## Problem 1: Disable beep when ACR122U NFC reader detects a tag

### Overview

By default NFC reader issues a beep as soon as the tag is close enough to the reader to be read, but BEFORE the software reads the tag. This is a problem, because the athlete hears a beep and wants to take off, but their tag has not yet been read.

Module: https://github.com/kate-holdener/splits/blob/main/src/readers/acr122u_nfc.py

Agent: Claude Code

Model: Sonnet 4.6

### Failed attempt 1

Defined two signals to send to the NFC Reader:
```
SET_BUZZER_OFF_APDU = [0xFF, 0x00, 0x52, 0x00, 0x00]
BEEP_APDU = [0xFF, 0x00, 0x40, 0x00, 0x04, 0x01, 0x01, 0x01, 0x01]
```

On each tag connection, in `_TagOvserver::update`, after tag connection is established, the `SET_BUZZER_OFF_APDU` is sent to the tag, then the tag is read, then `BEEP_APDU` is sent to the tag. However, at this point, the tag connection is already established and the initial beep had already happened. The agent claimed that we would still hear a pre-read beep on the first tag, but not on sugsequent tags. However, this claim was false, and every tag ended up issuing two beeps - one when the card was placed near the reader and the second one when the software read the card (because we now send `BEEP_APDU` after reading the card).


### Failed attempt 2

After pointing out that attempt 1 did not solve the problem, the agent attempted to communicate with the reader directly:
```
def _disable_auto_beep_on_reader(reader) -> None:                                 
    """Disable auto-beep at startup via a direct (no-card) connection so th     
      +e                                                                                 
    very first card presentation is also silent."""                             
    from smartcard.CardConnection import CardConnection                           
    try:                                                                          
        connection = reader.createConnection()                                    
        connection.connect(CardConnection.DIRECT)                                 
        connection.transmit(SET_BUZZER_OFF_APDU)                                  
        connection.disconnect() 
``` 
When running this code, the software issued a warning:
```
Warning: could not pre-disable beep: type object 'CardConnection' has no attribute  'DIRECT'                 
```
Then the agent imported
```
from smartcard.scard import SCARD_SHARE_DIRECT
```
and used
```
connection.connect(mode=SCARD_SHARE_DIRECT)
```
This resulted in a new warning:
```
 Warning: could not pre-disable beep: Invalid protocol in transmit: 
   must be CardConnection.T0_protocol, CardConnection.T1_protocol, or                        
  CardConnection.RAW_protocol
```

After this, the agenet attempted to use `control` to send commands directly to the reader hardware
```
ioctl = 0x003136B0 if sys.platform == "win32" else 0x42000D48
# ...
connection.control(ioctl, SET_BUZZER_OFF_APDU)
```
The warnings disappeared, but the initial beep, when the card is first placed near the reader remained.

### End result

After playing with the code some more, I got attempt 1 to work - there was still a beep on initial card detection, but future cards were silent. I ended up solving this with a different approach: keep the initial beep and issue a second beep when the card is read. So a double beep is a signal to the runner that they can take off.

## Problem 2 - performance optimization

### Overview

When an atlete is ready to run the next interval, they now scan their NFC tag and wait until they hear a double beep. The double beep happens pretty quickly, but I wanted to minimize athlete's wait time as much as possible. I asked Claude to: 
```
review my acr122u_nfc.py and see if I can speed up the reading of the nfc tag
```
Claude did a review and provided two suggestions:
```
  1. The callback fires after the beep (biggest win, easy fix)                               
  The event is only queued into self.queue after the beep APDU completes. That's an          
  unnecessary extra round-trip (~10–50 ms) of latency before anything downstream can act on  
  the scan. Reordering to: read UID → fire callback → then beep would shave that off.      
                                                                                             
  2. CardMonitor polling interval (potentially larger latency)                               
  pyscard's CardMonitoringThread polls every 300 ms by default. So on average you're waiting
  ~150 ms just for card presence to be detected before any APDU even gets sent. You can lower
   this by constructing the monitoring thread directly with a custom period.   
```

Suggestion 1 didn't really fix my problem, because I wanted to optimize athlete's wait time (wait for the double beep). This "fail" is due to my wording of the task - it was too ambiguous and the agent misinterpreted what I wanted to optimize. As a result, it provided a suggestion that made athlete's wait time a tiny bit longer.

### Failure
Suggestion 2 looked promising. I implemented the suggestion and tested the software. With time intervals this small, it was hard to tell if there was an improvement or not. So I tried setting the poll time to be 1 second (just to see if I notice the difference), and did not notice a change. I followed up with Claude asking it to find out the max value I can set the wait time (explaining that I am trying to notice the difference). After digging into the CardMonitoringThread code, Claude came back with:

```
This version of pyscard does not use a polling period at all. The CardMonitoringThread here uses CardRequest(timeout=60) and blocks on waitforcardevent() (line 165) — that's an event-driven call into the PC/SC stack that wakes up immediately when a card is physically detected. There is no PERIOD attribute, no sleep(), no polling loop.                                        

The CardMonitoringThread.PERIOD = 0.05 line I added does nothing — it just sets an unused class attribute. I need to remove it.
```

### End Result
No harm was done - an extra line of code that does nothing was added. However, Claude falsely led me to believe that the change was a performance improvement. 
