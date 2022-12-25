# target-STM32F401-drivers-SPI-IT
This project uses STM32CubeIDE and it's a program created to practice my C habilities during the course 'Mastering Microcontroller and Embedded Driver Development' from FastBit Embedded Brain Academy. I am using a NUCLEO-F401RE board.

## coding

The SPI interrupt requests can be found in 'Table 90'of the reference manual.

![image](https://user-images.githubusercontent.com/58916022/209451675-e631a6d0-f1fb-4a39-b7cd-bdee6000f3bb.png)

We create 2 more functions to send/receive data with the interruption routines (named as: SPI_SendDataWithIT(SPI_Handle_t *pSPIHandle, uint8_t *pTxBuffer, uint32_t Len) and SPI_ReceiveDataWithIT(SPI_Handle_t *pSPIHandle, uint8_t *pRxBuffer, uint32_t Len)). 

Then we copied the functions IRQInterruptConfig and IRQPriorityConfig from GPIO driver to SPI driver (with SPI_ prefix instead of GPIO_). Also, we modified the SPI_Handle_t structure.

![image](https://user-images.githubusercontent.com/58916022/209452111-f1414f5f-8e9e-4fea-9789-a3734383f517.png)

For the SPI_SendDataWithIT we need to: 

* 1. Save the Tx buffer address and Len information in some global variables. Those will be place inside the struct SPI_Handle_t.

* 2. Mark the SPI state as busy in transmission so that no other code can take over same SPI peripheral until tx is over.

* 3. Enable the TXEIE control bit to get interrupt whenever TXE flag is set in SR. This control bit is inside the CR2.

![image](https://user-images.githubusercontent.com/58916022/209453201-92996f0c-5f6f-4558-96d4-a9a0cd91a22d.png)

SPI Interruption can be triggered by the 3 interruption sources (seen in 'Table 90'). It can be due to RXNE flag (reception of data), TXE flag (transmition of data) or ERROR flag (for error).

To handle with the TXE interrupt we need to know if the transmit data is 8-bit or 16-bit, then write da byte(s) to the SPI data register (DR), decrement (one time for 1 byte e 2 times for 2 bytes) the Len-- variable. Then, if Len = 0, close the SPI Tx. Else, the transmission isn't over yet, we need to wait till another TXE interrupt.

For TXE interrupt we have the following code:

![image](https://user-images.githubusercontent.com/58916022/209470452-b91e6f67-5337-49ba-b1ed-e5f31b617097.png)

For RXNE interrupt we have the following code:

![image](https://user-images.githubusercontent.com/58916022/209470459-52777f72-0b5a-4d5c-910f-f5767f199f6a.png)

For ERROR interrupt we have the following code (NOT IMPLEMENTED: CRC ERROR, MODF [multi-master] and FRE [frame-format]):

OVR Implementation -> The OVR bit controls the generation of an interrupt when an error condition occurs (CRCERR, OVR, MODF in SPI mode). If data are received while the previously received data have not been read yet, an overrun is generated and the OVR flag is set. If the ERRIE bit is set in the SPI_CR2 register, an interrupt is generated to indicate the error. The new data is lost and the previous not read data remains in buffer.

![image](https://user-images.githubusercontent.com/58916022/209470465-52852849-3f0d-45b5-8ba2-ece3713d457e.png)

Those 3 functions we declared as helper functions that are private to SPI driver. We didn't declare those at the SPI.h file, instead we only declared in the SPI.c file with the key word 'static'. 'static' indicates that are actually private helper function (app cannot call them, compiler generates an error if so). We also added some atributes to those.

![image](https://user-images.githubusercontent.com/58916022/209470272-bd611ce3-b63d-40ac-bcaa-dd26a2b73ca1.png)

And at the end of this file we added those:

```
/// helper functions implementations
static void spi_txe_interrupt_handle(SPI_Handle_t *pSPIHandle){
	// Check the DFF  bit in CR1
	if((pSPIHandle->pSPIx->CR1 & (1 << SPI_CR1_DFF))){
		// 16-bit DFF
		//2.a. Load the data in to the DR
		pSPIHandle->pSPIx->DR = *((uint16_t*)pSPIHandle->pTxBuffer); //de-referenced value typecasted to 16 bit lenght
		pSPIHandle->TxLen--;
		pSPIHandle->TxLen--; // 2 bytes sent at time
		(uint16_t*)pSPIHandle->pTxBuffer++; // Increment the buffer
	}else{
		// 8-bit DFF
		pSPIHandle->pSPIx->DR = *((uint16_t*)pSPIHandle->pTxBuffer); //de-referenced value typecasted to 16 bit lenght
		pSPIHandle->TxLen--;
		(uint16_t*)pSPIHandle->pTxBuffer++; // Increment the buffer
	}

	if (!pSPIHandle->TxLen){ // if TxLen = 0, Tx is over
		SPI_CloseTransmission(pSPIHandle);
		SPI_ApplicationEventCallback(pSPIHandle,SPI_EVENT_TX_CMPLT);
	}
	//If Len isnt 0, the interruption is call again
}
static void spi_rxe_interrupt_handle(SPI_Handle_t *pSPIHandle){
	// do rxing as per the dff (8bit or 16bit)
	if(pSPIHandle->pSPIx->CR1 & (1 << 11)){
		//16 bit
		*((uint16_t*)pSPIHandle->pRxBuffer) = (uint16_t)pSPIHandle->pSPIx->DR;
		pSPIHandle->RxLen -= 2;
		pSPIHandle->pRxBuffer--;
		pSPIHandle->pRxBuffer--;
	}else{
		//8 bit
		*(pSPIHandle->pRxBuffer) = (uint8_t)pSPIHandle->pSPIx->DR;
		pSPIHandle->RxLen--;
		pSPIHandle->pRxBuffer--;
	}
	if (!pSPIHandle->RxLen){ // if RxLen = 0, Rx is over
		SPI_CloseReception(pSPIHandle);
		SPI_ApplicationEventCallback(pSPIHandle,SPI_EVENT_RX_CMPLT);
	}
}
static void spi_ovr_err_interrupt_handle(SPI_Handle_t *pSPIHandle){
	uint8_t temp;
	// 1. Clear the OVR flag (by reading DR and SR)
	if(pSPIHandle->TxState != SPI_BUSY_IN_TX){
		temp = pSPIHandle->pSPIx->DR;
		temp = pSPIHandle->pSPIx->SR;
	}
	(void)temp; // Compiler wont complain about unused variable
	// 2. Inform the application
	SPI_ApplicationEventCallback(pSPIHandle,SPI_EVENT_OVR_ERR);
}

```

The SPI_ApplicationEventCallback() was created as a week function. It is expected that a weak implementation will be override by another created by the user.

![image](https://user-images.githubusercontent.com/58916022/209470351-de234b7e-9ea4-4d26-8ee6-c8f932b1c23d.png)




