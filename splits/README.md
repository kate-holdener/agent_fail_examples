# Disable beep when ACR122U NFC reader detects a tag

## Overview
Repo: https://github.com/kate-holdener/splits

The software integrates with NFC reader hardware (ACR122U) and RFID scanner to monitor athletes' interval training, as they are running on the track. The NFC reader is used to detect the start time - an athlete scans their NFC tag before running off. By default NFC reader issues a beep as soon as the tag is close enough to the reader to be read, but BEFORE the software reads the tag. This is a problem, because the athlete hears a beep and wants to take off, but their tag has not yet been read.

Module: https://github.com/kate-holdener/splits/blob/main/src/readers/acr122u_nfc.py

Agent: Claude Code

Model: Sonnet 4.6

## Failed attempt 1

Defined two signals to send to the NFC Reader:
```
SET_BUZZER_OFF_APDU = [0xFF, 0x00, 0x52, 0x00, 0x00]
BEEP_APDU = [0xFF, 0x00, 0x40, 0x00, 0x04, 0x01, 0x01, 0x01, 0x01]
```

On each tag connection, in `_TagOvserver::update`, after tag connection is established, the `SET_BUZZER_OFF_APDU` is sent to the tag, then the tag is read, then `BEEP_APDU` is sent to the tag. However, at this point, the tag connection is already established and the initial beep had already happened. The agent claimed that we would still hear a pre-read beep on the first tag, but not on sugsequent tags. However, this claim was false, and every tag ended up issuing two beeps - one when the card was placed near the reader and the second one when the software read the card (because we now send `BEEP_APDU` after reading the card).


## Failed attempt 2

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

## End result

After playing with the code some more, I got attempt 1 to work - there was still a beep on initial card detection, but future cards were silent. I ended up solving this with a different approach: keep the initial beep and issue a second beep when the card is read. So a double beep is a signal to the runner that they can take off.
