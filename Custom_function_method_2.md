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

int srx_initiate(struct nfc_device *pnd, const nfc_modulation nm, nfc_target *pnt)
{
	/***** Commands *****/
	
	uint8_t initiate[] = "\x06\x00";
	uint8_t selectChip[] = "\xe0\x00";
	
	/***** Responses *****/
	
	uint8_t UIDBuffer[2]  = {0};
	uint8_t ChipBuffer[1] = {0};
	uint8_t readBuffer    =  0x00000000;
	
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
						  if ((res == NFC_ERFTRANS) && (CHIP_DATA(pnd)->last_status_byte == 0x01)) { // Chip timeout
							continue;
							printf("Continue\n");
						  }
						  else
						  return res;
					}
					printf("The chip id: %x\n", ChipBuffer[0]);
				}
				found = true;
				break;
			} while (pnd->bInfiniteSelect);
	}
	else printf("[-] Not the right type of tag ... \n");
	
	return 0;
}


```

This code will send the initiate command and get the CHIP ID from the tag/RFID. I tried to make it small as much as possible. This is where all our communications with the reader happen. I read and took commands from the SRx official doc. Here is the link: http://www.st.com/resource/en/datasheet/st25tb04k.pdf

5- Now we need to link these files together, we will use the same method used in "Method 1". We will open first "pn53x_uart.c":

Add this header: 

```C 
#include "chips/srx.h"
```

In the same file, add this code into the nfc_driver struct:

```C
.srx_rfid_initiate = srx_initiate,
```

6- Open the nfc-internal.h in libnfc folder and put this prototype:

```C
int (*srx_rfid_initiate)(struct nfc_device *pnd, const nfc_modulation nm, nfc_target *pnt);
```
7- Open now "nfc.h" in libnfc/include/nfc/ and this:

```C
NFC_EXPORT int nfc_srx_rfid_initiate(struct nfc_device *pnd, const nfc_modulation nm, nfc_target *pnt);
```
8- Open "nfc.c":

```C
int nfc_srx_rfid_initiate(struct nfc_device *pnd, const nfc_modulation nm, nfc_target *pnt)
{
	HAL(srx_rfid_initiate, pnd, nm, pnt);
}
```

If you didn't understand what I did just now go to the first method. I explained how libnfc works and why we have to do all of these steps.

9- Save everything and go to the terminal and type make && make install && make clean.
10- Open/create a main.c file and put this code:

```C
/**
 @Author: SAMIR Radouane
 @Date  : 31/03/2018
 
 *
 * This code is to read data blocks from STx card *
 **************************************************/


#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include <nfc/nfc.h>

#define RED   \033[0;31m
#define GREEN \033[1;32m
#define RESET \033[0m

#define MAX_TARGET_COUNT 16

void print_nfc_target(const nfc_target *pnt, bool verbose)
{
  char *s;
  str_nfc_target(&s, pnt, verbose);
  printf("%s", s);
  nfc_free(s);
}


void printhex (uint8_t *data, size_t bytes)
{
	size_t count = 0;
	for (count = 0; count < bytes; count++)
	printf("%02x ", data[count]);

	printf ("\n");
}

int main(void)
{
	nfc_device  *pnd;
	nfc_target   nt;

	nfc_context *context;

	/** initiate libnfc and set nfc_context **/
	nfc_init(&context);
	if (context == NULL) {
		printf("\033[0;31m[-]\033[0m Error loading libnfc, must quit ... \n");
		exit(EXIT_FAILURE);
	}

	/** Get libnfc Version **/
	const char *libnfc_version = nfc_version();
	printf("\033[1;32m[*]\033[0m Libnfc version: %s\n", libnfc_version);

	/** Open NFC Device **/
	pnd = nfc_open(context, NULL);
	if (pnd == NULL) {
		printf("\033[0;31m[-]\033[0m Couldn't open the NFC device, must quit ...\n");
		exit(EXIT_FAILURE);
	}

	//Initiate the device
	if (nfc_initiator_init(pnd) < 0) {
		printf("\033[0;31m[-]\033[0m Error to initiate the NFC device, must quit...\n");
		exit(EXIT_FAILURE);
	}

	//Everything is all right, so now is opened ...
	printf("\033[1;32m[*]\033[0m The device is opened: %s\n", nfc_device_get_name(pnd));

	/**=======================================================================**/
	/* This is a known bug from libnfc. To read 14443BSR you should initiate
	 * ISO14443B to prepare register for PN532 and after you can call
	 * SRx function.
	 */

	nfc_target anttemp[1];
	nfc_modulation nmtemp;
    	nmtemp.nmt = NMT_ISO14443B;
    	nmtemp.nbr = NBR_106;
	nfc_initiator_list_passive_targets(pnd, nmtemp, anttemp, MAX_TARGET_COUNT);
        
	/**=========================================================================**/

	/** Choose the type you wanna read **/
	const nfc_modulation SRx = {
		.nmt = NMT_ISO14443B2SR,
		.nbr = NBR_106,
	};

	nfc_target ant[1];
	nfc_srx_rfid_initiate(pnd, SRx, anttemp);

	nfc_close(pnd);
	nfc_exit(context);
	exit(EXIT_SUCCESS);
	return 0;
}

```

11 - Compile everything using this command:

> gcc -o nfc_SRx main.c  -I../libnfc  -I../libnfc/include -lnfc

May be You have to change the folder path in -I parameters so be careful with this.

12 - execute the program:
> ./nfc_SRx

The program will show a CHIP ID Byte, it will change each time you run the program.

Let me know if you struggle to complete this tutorial.

Thanks.




