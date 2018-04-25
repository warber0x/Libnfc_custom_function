# How to add a custom function to libnfc: Method 2

If you read the first method on how to add a custom function, this will be a piece of cake. The difference between the first and this method is to create two additional files which will contain our custom code.
These files we will call them srx.c and srx.h. Everything we will need to add we'll put them there.

Follow these steps and everything will be all right:

1- Go to the path of libnfc library and go to: libnfc/chips/

2- Create a file "srx.h":

3- Open the file with your favorite editor and put this code:

```C
#include <nfc/nfc-types.h>
#include "pn53x-internal.h"

int srx_initiate(struct nfc_device *pnd);
```

4- In the same folder, create another file "srx.c" and put this code:

```C
#ifdef HAVE_CONFIG_H
#  include "config.h"
#endif // HAVE_CONFIG_H

#include <inttypes.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdlib.h>

#include "nfc/nfc.h"
#include "nfc-internal.h"
#include "srx.h"
#include "pn53x-internal.h"

#include "mirror-subr.h"

#define LOG_CATEGORY "libnfc.chip.pn53x"
#define LOG_GROUP NFC_LOG_GROUP_CHIP

int srx_initiate(struct nfc_device *pnd)
{
/***** Commands *****/
	
	uint8_t initiate[] = "\x06\x00";
	uint8_t selectChip[] = "\xe0\x00";
	uint8_t getUID = "\x0b";
	uint8_t readBlock[] = "\x08";
	uint8_t writeBlock[] = "\x09";
	
	/***** Responses *****/
	
	uint8_t UIDBuffer[2] = {0};
	uint8_t ChipBuffer[1]    =   {0};
	uint8_t readBuffer    =   0x00000000;
	
	int res = 0;
	bool found = false;
	if (nm.nmt == NMT_ISO14443B2SR)
	{
			if (CHIP_DATA(pnd)->type == RCS360) 
			{
				// TODO add support for RC-S360, at the moment it refuses to send raw frames without a first select
				pnd->last_error = NFC_ENOTIMPL;
				return pnd->last_error;
			}
			// No native support in InListPassiveTarget so we do discovery by hand
			if ((res = nfc_device_set_property_bool(pnd, NP_FORCE_ISO14443_B, true)) < 0) {
			  return res;
			}
			if ((res = nfc_device_set_property_bool(pnd, NP_FORCE_SPEED_106, true)) < 0) {
			  return res;
			}
			if ((res = nfc_device_set_property_bool(pnd, NP_HANDLE_CRC, true)) < 0) {
			  return res;
			}
			if ((res = nfc_device_set_property_bool(pnd, NP_EASY_FRAMING, false)) < 0) {
			  return res;
			}
			
			do 
			{
				if (nm.nmt == NMT_ISO14443B2SR) 
				{
					if ((res = pn53x_initiator_transceive_bytes(pnd, initiate, 2, ChipBuffer, sizeof(ChipBuffer), 0)) < 0) 
					{
						  if ((res == NFC_ERFTRANS) && (CHIP_DATA(pnd)->last_status_byte == 0x01))
							continue;
						  else
						  return res;
					}
					printf("The chip ID is: %x\n", ChipBuffer[0]);
				}
				found = true;
				break;
			} while (pnd->bInfiniteSelect);
	}
	else printf("[-] Not the right type of tag ... \n");
	
	return 0;
}


```

This code will send the initiate command and get the CHIP ID for the tag/RFID

5- Now we need to link these files to the core, we will use the same method used in "Method 1". We will open first "pn53x_uart.c":

Add this header: 

```C 
#include "chips/srx.h"
```

And this code into the nfc_driver struct:

```C
.srx_rfid_initiate = srx_initiate,
```

6- Open the nfc-internal.h in libnfc folder and put this prototype:
```C
int srx_rfid_initiate(...)
```



