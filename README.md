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

![image](https://user-images.githubusercontent.com/58916022/209453768-0d2065d5-23a3-4ede-8285-1c052d00fba4.png)

For RXNE interrupt we have the following code:

![image](https://user-images.githubusercontent.com/58916022/209453773-c2573b57-cfbd-4259-bd86-1f23616d8a21.png)

For ERROR interrupt we have the following code (NOT IMPLEMENTED: CRC ERROR, MODF [multi-master] and FRE [frame-format]):

OVR Implementation -> The OVR bit controls the generation of an interrupt when an error condition occurs (CRCERR, OVR, MODF in SPI mode). If data are received while the previously received data have not been read yet, an overrun is generated and the OVR flag is set. If the ERRIE bit is set in the SPI_CR2 register, an interrupt is generated to indicate the error. The new data is lost and the previous not read data remains in buffer.

![image](https://user-images.githubusercontent.com/58916022/209453776-5c861231-d19c-4e73-9f42-98d91dc28d6f.png)


