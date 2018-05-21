# How to add a custom function to libnfc: Method 1

Whenever you want to create a custom function in libnfc to add a new feature or read a tag/RFID Card that is not fully integrated (like SRx for my case), this article is for you:

That's what I'm using:
- Arduino UNO.
- Adafruit PN532 
- Breadboard
- PN532 soldered to use SPI protocol
- Arduino & libnfc configured to use UART through Arduino.


1 - First, you have to know if you will use Arduino like a bridge to adafruit PN532 or not. For my case, I configured libnfc like so:
> autoreconf -vis
> ./configure --with-drivers=pn532_uart --sysconfdir=/etc --prefix=/usr
> make clean all

If you have a specefic needs, you can visit and follow the tutorial from nfc-tools.org to learn how to compile your libnfc.

2 - When the libnfc is installed and the examples work, open pn53x.h in libnfcfolder/libnfc/chips/
3 - Add your prototype function, in my case, I try to add a custom function that will read SRx rfid TAG. At first, I will add a copy of function that's responsible to initiate the tag and read the UID:

```C

/**************************** My custom prototype  RED1 *********************************/
int    pn53x_initiator_SRx(struct nfc_device *pnd,
                                             const nfc_modulation nm,
                                             const uint8_t *pbtInitData, const size_t szInitData,
                                             nfc_target *pnt);
/*********************************************************************************/

```

4 - Now go to the pn53x.c in the same folder as the pn53x.h and add function definition:

```C

/** ** ** ** ** ** ** ** ** ** ** ** ** ** ** ** ** ** ** *
 * 	My custom function                                *
 ** ** ** ** ** ** ** ** ** ** ** ** ** ** ** ** ** ** ** */
static int pn53x_initiator_SRx_ext(struct nfc_device *pnd,
                                          const nfc_modulation nm,
                                          const uint8_t *pbtInitData, const size_t szInitData,
                                          nfc_target *pnt,
                                          int timeout)
{
		uint8_t  abtTargetsData[PN53x_EXTENDED_FRAME__DATA_MAX_LEN];
		size_t  szTargetsData = sizeof(abtTargetsData);
		int res = 0;
		nfc_target nttmp;

		if (nm.nmt == NMT_ISO14443BI || nm.nmt == NMT_ISO14443B2SR || nm.nmt == NMT_ISO14443B2CT) 
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
			bool found = false;
			
		do {
				if (nm.nmt == NMT_ISO14443B2SR) 
				{
					// Some work to do before getting the UID...
					uint8_t abtInitiate[] = "\x06\x00";
					size_t szInitiateLen = 2;
					uint8_t abtSelect[] = { 0x0e, 0x00 };
					uint8_t abtRx[1];
					// Getting random Chip_ID
					if ((res = pn53x_initiator_transceive_bytes(pnd, abtInitiate, szInitiateLen, abtRx, sizeof(abtRx), timeout)) < 0) 
					{
						if ((res == NFC_ERFTRANS) && (CHIP_DATA(pnd)->last_status_byte == 0x01)) 
						continue;
						else
						return res;
					}
					
					abtSelect[1] = abtRx[0]; // abtRx contains CHIP_ID
					if ((res = pn53x_initiator_transceive_bytes(pnd, abtSelect, sizeof(abtSelect), abtRx, sizeof(abtRx), timeout)) < 0) 
					return res;
				} 
			  
			  if ((res = pn53x_initiator_transceive_bytes(pnd, pbtInitData, szInitData, abtTargetsData, sizeof(abtTargetsData), timeout)) < 0) 
			  {
					if ((res == NFC_ERFTRANS) && (CHIP_DATA(pnd)->last_status_byte == 0x01)) 
					continue;
					else
					return res;
			  }
			  
			  nttmp.nm = nm;
			  if ((res = pn53x_decode_target_data(abtTargetsData, szTargetsData, CHIP_DATA(pnd)->type, nm.nmt, &(nttmp.nti))) < 0) 
			  return res;
			  
			  found = true;
			  break;
			} while (pnd->bInfiniteSelect);
			
			if (! found)
			return 0;
		} 
		else
		{
			const pn53x_modulation pm = pn53x_nm_to_pm(nm);
			if (PM_UNDEFINED == pm) 
			{
			  pnd->last_error = NFC_EINVARG;
			  return pnd->last_error;
			}

			if ((res = pn53x_InListPassiveTarget(pnd, pm, 1, pbtInitData, szInitData, abtTargetsData, &szTargetsData, timeout)) <= 0)
			  return res;

			if (szTargetsData <= 1) // For Coverity to know szTargetsData is always > 1 if res > 0
			  return 0;

			nttmp.nm = nm;
			if ((res = pn53x_decode_target_data(abtTargetsData + 1, szTargetsData - 1, CHIP_DATA(pnd)->type, nm.nmt, &(nttmp.nti))) < 0) 
			return res;
		}
		  
		  if (pn53x_current_target_new(pnd, &nttmp) == NULL) 
		  {
			pnd->last_error = NFC_ESOFT;
			return pnd->last_error;
		  }
		  // Is a tag info struct available
		  if (pnt) 
		  memcpy(pnt, &nttmp, sizeof(nfc_target));
		  
		  return abtTargetsData[0];
}

int pn53x_initiator_SRx(struct nfc_device *pnd,
                                      const nfc_modulation nm,
                                      const uint8_t *pbtInitData, const size_t szInitData,
                                      nfc_target *pnt)
{
  return pn53x_initiator_SRx_ext(pnd, nm, pbtInitData, szInitData, pnt, 0);
}

/************************ End of Custom code ***************************/

```

As you noticed, there are two functions:
- pn53x_initiator_SRx_ext
- pn53x_initiator_SRx

The second one is the function that exists in the "pn53x.c" that calls the first, the first one contains all the logic but it doesn't exists in the "pn53x.h" 

5- now go to nfc.h and add these prototypes:

```C

/** My custom code RED1  **/
NFC_EXPORT int nfc_initiator_SRx_targets(nfc_device *pnd, const nfc_modulation nm, const uint8_t *pbtInitData, const size_t szInitData, nfc_target *pnt);
NFC_EXPORT int nfc_initiator_SRx(nfc_device *pnd, const nfc_modulation nm, nfc_target ant[], const size_t szTargets);
/** End my custom code     **/

```

The function "nfc_initiator_SRx" is what the programmer will use and call in his code, when used, it calls  "nfc_initiator_SRx_targets"  with a little custom code added before we call the desired function. The function is responsible to call what can control RFID reader. All core function reside in "pn53x.c", for us we are concerned about "pn53x_initiator_SRx". The function communicate through a struct responsible to make calls between the low level function (pn53x.c) and high level functions (nfc.c)

6- Open the nfc.c and add these functions:

```C

/***  Custom code RED1 *****/

int  nfc_initiator_SRx_targets(nfc_device *pnd, const nfc_modulation nm, const uint8_t *pbtInitData, const size_t szInitData, nfc_target *pnt)
{
	uint8_t *abtInit = NULL;
  uint8_t abtTmpInit[MAX(12, szInitData)];
  size_t  szInit = 0;
  if (szInitData == 0) {
    // Provide default values, if any
    prepare_initiator_data(nm, &abtInit, &szInit);
  } else if (nm.nmt == NMT_ISO14443A) {
    abtInit = abtTmpInit;
    iso14443_cascade_uid(pbtInitData, szInitData, abtInit, &szInit);
  } else {
    abtInit = abtTmpInit;
    memcpy(abtInit, pbtInitData, szInitData);
    szInit = szInitData;
  }

  HAL(initiator_SRx, pnd, nm, abtInit, szInit, pnt);
}

int nfc_initiator_SRx(nfc_device *pnd,
                                   const nfc_modulation nm,
                                   nfc_target ant[], const size_t szTargets)
{
  nfc_target nt;
  size_t  szTargetFound = 0;
  uint8_t *pbtInitData = NULL;
  size_t  szInitDataLen = 0;
  int res = 0;

  pnd->last_error = 0;

  // Let the reader only try once to find a tag
  bool bInfiniteSelect = pnd->bInfiniteSelect;
  if ((res = nfc_device_set_property_bool(pnd, NP_INFINITE_SELECT, false)) < 0) {
    return res;
  }

  prepare_initiator_data(nm, &pbtInitData, &szInitDataLen);

  while (nfc_initiator_SRx_targets(pnd, nm, pbtInitData, szInitDataLen, &nt) > 0) {
    size_t i;
    bool seen = false;
    // Check if we've already seen this tag
    for (i = 0; i < szTargetFound; i++) {
      if (memcmp(&(ant[i]), &nt, sizeof(nfc_target)) == 0) {
        seen = true;
      }
    }
    if (seen) {
      break;
    }
    memcpy(&(ant[szTargetFound]), &nt, sizeof(nfc_target));
    szTargetFound++;
    if (szTargets == szTargetFound) {
      break;
    }
    nfc_initiator_deselect_target(pnd);
    // deselect has no effect on FeliCa and Jewel cards so we'll stop after one...
    // ISO/IEC 14443 B' cards are polled at 100% probability so it's not possible to detect correctly two cards at the same time
    if ((nm.nmt == NMT_FELICA) || (nm.nmt == NMT_JEWEL) || (nm.nmt == NMT_ISO14443BI) || (nm.nmt == NMT_ISO14443B2SR) || (nm.nmt == NMT_ISO14443B2CT)) {
      break;
    }
  }
  if (bInfiniteSelect) {
    if ((res = nfc_device_set_property_bool(pnd, NP_INFINITE_SELECT, true)) < 0) {
      return res;
    }
  }
  return szTargetFound;
}

/******** ****** End code ******* *********/

```

These functions prepare the ground and what should be called. The "nfc_initiator_SRx_targets"  makes the link between the low level functions through "HAL" macro in "nfc_internal.h". I will explain in details all of these because it's worth it (The devil is in the details ;)):

This is the macro:

```C
// the macro that points to desired function
#define HAL( FUNCTION, ... ) pnd->last_error = 0; \
  if (pnd->driver->FUNCTION) { \
    return pnd->driver->FUNCTION( __VA_ARGS__ ); \

```

7 - once you added all of these, we will link all these together. Open the file "nfc-internal.h" in libnfc folder and add this line as member of the struct "nfc_driver"  :

```C

int (*initiator_SRx) (struct nfc_device *pnd,  const nfc_modulation nm,  const uint8_t *pbtInitData, const size_t szInitData, nfc_target *pnt);

```

After that, open the file pn53x_uart.c in libnfc/drivers path if of course you're using arduino and adafruit PN532 through UART. Add this line in the nfc_driver struct:
  
  ```C
  
  /****** My custom code RED1 ********/
  .initiator_SRx = pn53x_initiator_SRx,
  /***********************************/
  
  ```

The ".initiator_SRx" name must be the same that was used in nfc.c as well as the "pn53x_initiator_SRx". If your  final program doesnt work. Probably is due to the lack of information in nfc_driver struct. Why is that ?

Let me now explain to you how this library works:
Till now, you are aware that there are two files that contains all of the functions responsible to communicate with our device: nfc.c and pn53x.c. These files are linked together through a struct called nfc_driver. This struct has a multiple functions pointers as members, these members will point to the name of the low level function that will be excuted. For instance:

This is an initialization of the struct

```C

const struct nfc_driver pn532_uart_driver = {

  .initiator_init                   = pn53x_initiator_init,
  .initiator_init_secure_element    = pn532_initiator_init_secure_element,
  .initiator_select_passive_target  = pn53x_initiator_select_passive_target,
  .initiator_poll_target            = pn53x_initiator_poll_target,
  .initiator_select_dep_target      = pn53x_initiator_select_dep_target,
  .initiator_deselect_target        = pn53x_initiator_deselect_target,
  .initiator_transceive_bytes       = pn53x_initiator_transceive_bytes,
  .initiator_transceive_bits        = pn53x_initiator_transceive_bits,
  .initiator_transceive_bytes_timed = pn53x_initiator_transceive_bytes_timed,
  .initiator_transceive_bits_timed  = pn53x_initiator_transceive_bits_timed,
  .initiator_target_is_present      = pn53x_initiator_target_is_present,
  
  /****** My custom code RED1 ********/
  .initiator_SRx = pn53x_initiator_SRx,
  /***********************************/
  
};

```
This is the struct itself:

```C
struct nfc_driver {



  int (*initiator_transceive_bits_timed)(struct nfc_device *pnd, const uint8_t *pbtTx, const size_t szTxBits, const uint8_t *pbtTxPar, uint8_t *pbtRx, uint8_t *pbtRxPar, uint32_t *cycles);
  int (*initiator_target_is_present)(struct nfc_device *pnd, const nfc_target *pnt);
  
  /** Custom Code RED1 **/
  int (*initiator_SRx) (struct nfc_device *pnd,  const nfc_modulation nm,  const uint8_t *pbtInitData, const size_t szInitData, nfc_target *pnt);
  int (*read_block_SRx)(struct nfc_device *pnd, const nfc_modulation nm, nfc_target *pnt, uint32_t address);)
  /******* ** ** * ** ** * * */

};

```
As you noticed, "initiator_SRx" is the member that I added in nfc_driver struct as well as in PN532_uart.c, The name I gave is the name of my custom function in pn53x.c. WOOOOFF!!! It's little bit tricky but if you pay attention to these files you surely will understand.

8 - That's all what we can do for now. The next thing to do is to excute the make command, you "cd" to the libnfc folder and do "make", "make install" and "make clean" to recompile everything and to have a new fresh library.

9 - Create a new file called "main.c" and put this code in it:

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

	/** Choose the type you wanna read **/
	const nfc_modulation SRx = {
		.nmt = NMT_ISO14443B2SR,
		.nbr = NBR_106,
	};

	nfc_target ant[1];
	
	if (  nfc_initiator_SRx(pnd, SRx, ant, MAX_TARGET_COUNT) > 0)
	{
		printf("[+] Card SRx detected ... \n");
		printf("[+] Reading UID ... \n");
		printf("[+] DATA:  ");
		print_nfc_target(&ant[0], 0);
	}

	nfc_close(pnd);
	nfc_exit(context);
	exit(EXIT_SUCCESS);
	return 0;
}

```

10 - Compile everything using this command:

> gcc -o nfc_SRx main.c  -I../libnfc  -I../libnfc/include -lnfc

May be You have to change the folder path in -I parameters so be careful with this.

11 - execute the program:
> ./nfc_SRx

It should show you the UID of the rfid tag you put in your reader.

Have fun ...


