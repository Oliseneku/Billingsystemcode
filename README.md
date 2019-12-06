// CONFIG1
#pragma config FOSC = XT        // Oscillator Selection bits (XT oscillator: Crystal/resonator on RA6/OSC2/CLKOUT and RA7/OSC1/CLKIN)
#pragma config WDTE = OFF       // Watchdog Timer Enable bit (WDT disabled and can be enabled by SWDTEN bit of the WDTCON register)
#pragma config PWRTE = OFF      // Power-up Timer Enable bit (PWRT disabled)
#pragma config MCLRE = ON       // RE3/MCLR pin function select bit (RE3/MCLR pin function is MCLR)
#pragma config CP = OFF         // Code Protection bit (Program memory code protection is disabled)
#pragma config CPD = OFF        // Data Code Protection bit (Data memory code protection is disabled)
#pragma config BOREN = ON       // Brown Out Reset Selection bits (BOR enabled)
#pragma config IESO = ON        // Internal External Switchover bit (Internal/External Switchover mode is enabled)
#pragma config FCMEN = OFF      // Fail-Safe Clock Monitor Enabled bit (Fail-Safe Clock Monitor is disabled)
#pragma config LVP = OFF        // Low Voltage Programming Enable bit (RB3 pin has digital I/O, HV on MCLR must be used for programming)

// CONFIG2
#pragma config BOR4V = BOR40V   // Brown-out Reset Selection bit (Brown-out Reset set to 4.0V)
#pragma config WRT = OFF        // Flash Program Memory Self Write Enable bits (Write protection off)

//#include <xc.h>
#include <pic16f886.h>
//#include <math.h>
#include "prepaid.h"  //PORTEbits.RE1 = 1;
#define _XTAL_FREQ 8000000

#define Process_LED PORTBbits.RB0
#define Amp_in PORTBbits.RB6
#define Volt_in PORTBbits.RB4

struct{
    unsigned TB0 : 1;
    unsigned TB1 : 1;
    unsigned TB2 : 1; // operational test bits
    unsigned TB3 : 1;
    unsigned TB4 : 1;
    unsigned TB5 : 1;
    unsigned TB6 : 1;
    unsigned TB7 : 1;
}test_characters;

#define zerovoltage_latch test_characters.TB2 // operational test bits
#define zerocurrent_latch test_characters.TB3
#define V_signal test_characters.TB4
#define I_signal test_characters.TB5

unsigned char time_to_update; //Might change to a single bit
unsigned char reload;
unsigned char action_select;
unsigned char parse; //PORTBbits.RB7 = 1;

float power_div;

void interrupt service(void) {
    //TIMER 1 INTERRUPT FUNCTIONS...............................................
    if (PIR1bits.TMR1IF == 1){
        if(stop_clock == 2 && realUnits > 0){ //When 1sec has elapsed
            time_to_update = 4; //Starts Synchronous updates
            stop_clock = 0;
        }else stop_clock++;
        TMR1H = 11; //Timer 1 registers for timing configurations
        TMR1L = 219;
    PIR1bits.TMR1IF = 0;    
    } 
    //USART RX FUNCTIONS........................................................
    if(PIR1bits.RCIF == 1){
        Process_LED = 1;
        fix();
        //=======================================================RCREG Functions
            if (RCREG == byte_message) {
                ql++;
                if (ql == word_size) {
                    if (word_size == 5 && index == 3) {
                        CMD = 3;
                    }
                    index++;
                    ql = 0;
                }
                garbage = RCREG;
            } else ql = 0;
            if (RCREG == second_byte_message) {//DEV route.
                LM++;
                if (LM == second_word_size) {
                    if (second_word_size == 4) {
                    CMD = 5;
                    index = 0; //Breaks packet reception
                    exx = 0;
                    LM = 0;
                }
                if(second_word_size == 6){
                       //sig = 7;//Signifies that we are taking time and date check route instead.
                       index++;
                }
                }
            } else LM = 0;
            if (RCREG == third_byte_message) {//Save number route.
                dd++;
                if (dd == third_word_size) {
                    CMD = 7;
                    index = 0; //Breaks packet reception.
                    exx = 0;
                    dd = 0;
                }
            } else dd = 0;
            if (RCREG == forth_byte_message) {//Save number route.
                fif++;
                if (fif == forth_word_size) {
                    CMD = 9;
                    index = 0; //Breaks packet reception.
                    exx = 0;
                    fif = 0;
                }
            } else fif = 0;
        //======================================================================End of RCREG
            switch (index) {
                case 2:reg_count = 15;
                       action_select = store_in_RAM();
                       
                  /*     if(sig!=7){ //normal route
                           acquire = store_in_RAM(0,229);
                       }else if(sig == 7){  //clock check route
                           acquire = store_in_RAM(7,255); //return value not needed by reg_sms
                       }*/
                       
                    break;
                case 4:reg_count = 25; //was set to 25 to avoid reaching this value.
                       action_select = store_in_RAM(); //might remove action_select.
                    break;
            }
        PIR1bits.RCIF = 0;
        Process_LED = 0;
    }
}

void main (){
ANSEL = 0;
ANSELH = 0;
INTCON = 192;
PIE1 = 1; //For Timer1 interrupt.
PORTA = PORTC = PORTB = 0;
TRISA = 32; //0;//4;
TRISB = 88; //80;
TRISC = 128;//131;
//IOCB = 80;
sample = MUTE;
ASSERT = 0;
Lcd_Start();
Lcd_home_screen(); 
my_init_usart(9600);
__delay_ms(300);
T1CON = 49; //Was on 48. Timer1 configurations. T1CON |= 1; //Enables TImer 1 included
TMR1H = 11; //Timer 1 registers for timing configurations
TMR1L = 219; // 3035 counts(From 62500 to 65535) 0.5secs
my_ADC_Init();
//Write_eeprom(39, 4); Balance
//Write_eeprom(20, 1); //past assert state
//Write_eeprom(22, 0); //error code 
parse = ql = LM = dd = 0; //dev = fif = 0;
exx = index = 0;
display_permit = time_to_update = 0;
CMD = stop_clock = 0;

realUnits = ReadDecimalFraction(); //Read units value from EPROM.
relay_state = Read_eeprom(254); //Read relay_state value from EPROM
if (relay_state == ACTIVE){ // Active relay state cuts off power supply.
    activate_deactivate(2);//Activate.
    display_permit = 2;
}else activate_deactivate(4);//Deactivate.

while(1) {
    //UPDATE LCD ROUTE..........................................................
    switch (display_permit){
        case 0: Lcd_Clear();
                Lcd_Set_Cursor(1, 1);//Avoids continuously loading these characters to the LCD. Might remove this
                Lcd_Print_String(UNITS_LABEL); //Make sure this stay on the screen once system is idle
                Lcd_Set_Cursor(2, 1);
                Lcd_Print_String(LOAD_LABEL); 
                display_permit = 1;
            break;
        case 2: Lcd_Clear();
                Lcd_Set_Cursor(1, 1);//Avoids continuously loading these characters to the LCD. Might remove this
                Lcd_Print_String(R_UNI); //Make sure this stay on the screen once system is idle
                display_permit = 3;
            break;
    }
    
    //SYNCHRONOUS UPDATE ROUTE..................................................
    if(time_to_update == 4){
        PORTBbits.RB7 = ~PORTBbits.RB7;
        tamper_routine();
        if(relay_state == INACTIVE){
        //Power measurement
        load_monitoring();
        my_inttoStr(power); //Load demand monitoring mode (KVA).
        Lcd_Set_Cursor(2, 9);
        Lcd_Print_String(ERASE); //Wipe off displayed characters 
        Lcd_Set_Cursor(2, 9);
        Lcd_Print_String(meter); 
        power_div = power; //Pass power to type float for proper handling 
        power_div /= 3600 ; // 0.00027777 = 1/3600. Converts to Watt minutes.
        //realUnits = ReadDecimalFraction(); IRRELEVANT
        realUnits *= 1000; //60*1000. Watt minutes.
        
        if(PORTBbits.RB3 == 1){ //debugging feature.
            power_div = 200;
        }
        
        realUnits -= (float) power_div;
        realUnits /= 1000;
        StoreDecimalFraction(realUnits);
        //PORTBbits.RB7 = ~PORTBbits.RB7;
        displayUnits = (int) realUnits; //convert to type integer and assign
        my_inttoStr(displayUnits); //Load demand monitoring mode (KVA).
        Lcd_Set_Cursor(1, 7);
        Lcd_Print_String(ERASE);
        Lcd_Set_Cursor(1, 7);
        Lcd_Print_String(meter); //Wipe off displayed characters 
        
        if(realUnits <= 0){ //if Units has been exhausted.
            activate_deactivate(2);//Deactivate.
            parse = swift = 0;
            while (swift < 14){
                phone_num[swift] = Read_eeprom(swift + 1);
                //phone_no[swift] = parse;
                swift++;
            }
            send_message(phone_num, 'R'); //82. Notify user through the saved number
        }
        }
        time_to_update = 0;
    }   
    
    //RELOAD ROUTE..............................................................
    if (action_select == 3){
        Process_LED = 1;
        reloading();
        if (realUnits > 0) { // Runs after UNits has been reloaded. Was displayUnits > 0 before.
                activate_deactivate(4); //Activate
            }
        action_select = 0;
        Process_LED = 0;
    }
    
    //SAVE NUMBER ROUTE..............................................................
    if (CMD == 7){
        Process_LED = 1;
        swift = action_select = 0; //action select was reused here.
        while(action_select < 14){
            STATUSbits.IRP = 1;
            FSR = 289 + action_select; //RAM address
            swift = INDF;
            STATUSbits.IRP = 0;
            Write_eeprom(action_select + 1, swift);
            phone_num[action_select] = swift;
            
            action_select++;
        }
        send_message(phone_num, 'W'); //87
        action_select = 0;
        CMD = 0;
        Process_LED = 0;
    }
    
    //BALANCE CHECK ROUTE..............................................................
    if (CMD == 9){
        Process_LED = 1;
        parse = swift = 0;
            while (swift < 14){
                STATUSbits.IRP = 1;
                FSR = 289 + swift; //RAM address
                parse = INDF;
                STATUSbits.IRP = 0;
                phone_num[swift] = parse;
                swift++;
            }
            my_inttoStr(displayUnits);
            send_message(phone_num, 'T'); //82. Notify user through the saved number
        CMD = 0;
        Process_LED = 0;
    }
}
}
//Prepaid meter methods:
#include <xc.h>
#include <pic16f886.h>
#include "prepaid.h"
#define _XTAL_FREQ 8000000

#define relay_control PORTBbits.RB1
#define RS PORTCbits.RC0
#define EN PORTCbits.RC1

//#define D4 PORTCbits.RC2
//#define D5 PORTCbits.RC3
//#define D6 PORTCbits.RC4
//#define D7 PORTCbits.RC5

//====================================HANDLES LCD FUNCTIONS.....................
void Lcd_SetBit(char data_bit) { //Based on the Hex value Set the Bits of the Data Lines
    PORTC &= 195; //clearing the required PORTC bits
    PORTC |= data_bit << 2;
}

void Lcd_Cmd(char a) {
    RS = 0;
    Lcd_SetBit(a); //Incoming Hex value
    EN = 1;
    __delay_ms(4);
    EN = 0;
}

void Lcd_Clear(void) {
    Lcd_Cmd(0); //Clear the LCD
    Lcd_Cmd(1); //Move the cursor to first position
}

void Lcd_Set_Cursor(char u, char b) {
    char temp, z, y;
    if (u == 1) {
        temp = 0x80 + b - 1; //80H is used to move the cursor
        z = temp >> 4; //Lower 8-bits
        y = temp & 0x0F; //Upper 8-bits
        Lcd_Cmd(z); //Set Row
        Lcd_Cmd(y); //Set Column
    } else if (u == 2) {
        temp = 0xC0 + b - 1;
        z = temp >> 4; //Lower 8-bits
        y = temp & 0x0F; //Upper 8-bits
        Lcd_Cmd(z); //Set Row
        Lcd_Cmd(y); //Set Column
    }
}

void Lcd_Start(void) {
    int i;
    Lcd_SetBit(0x00);
    //for (i = 1065244; i <= 0; i--)
    //NOP();
    Lcd_Cmd(0x03);
    __delay_ms(5);
    Lcd_Cmd(0x03);
    __delay_ms(11);
    Lcd_Cmd(0x03);
    Lcd_Cmd(0x02); //02H is used for Return home -> Clears the RAM and initializes the LCD
    Lcd_Cmd(0x02); //02H is used for Return home -> Clears the RAM and initializes the LCD
    Lcd_Cmd(0x08); //Select Row 1
    Lcd_Cmd(0x00); //Clear Row 1 Display
    Lcd_Cmd(0x0C); //Select Row 2
    Lcd_Cmd(0x00); //Clear Row 2 Display
    Lcd_Cmd(0x06);
}

void Lcd_Print_Char(char da_ta, int delay_print) { //Send 8-bits through 4-bit mode
    int i;
    char Lower_Nibble, Upper_Nibble;
    Lower_Nibble = da_ta & 0x0F;
    Upper_Nibble = da_ta & 0xF0;
    RS = 1; // => RS = 1
    Lcd_SetBit(Upper_Nibble >> 4); //Send upper half by shifting by 4
    EN = 1;
    for (i = delay_print; i >= 0; i--); 
    //NOP();
    EN = 0;
    Lcd_SetBit(Lower_Nibble); //Send Lower half
    EN = 1;
    for (i = delay_print; i >= 0; i--); 
    //NOP();
    EN = 0;
}

void Lcd_Print_String(char *a) {
    int ik;
    for (ik = 0; a[ik] != '\0'; ik++) {
        Lcd_Print_Char(a[ik], 1); //Split the string using pointers and call the Char function 
    }
}

void Lcd_home_screen(void){
    Lcd_Clear();
    Lcd_Set_Cursor(1, 1);
    Lcd_Print_String("PREPAID METER!!!"); //Make sure this stay on the screen once system is idle
}

void L_R_TextScroll(unsigned char *string_select, unsigned short string_length) {
    //Imagine the characters are viewed through a window with adjustable width
    //The left window pane is "place", while that of the right is "adjust"
    //The width increases as "adjust" sweeps across and becomes constant as "place" begins to move at the same rate.
    //The right window pane "adjust" finally gets to the end, thus reducing the window's width as "place" also approaches the end.
    unsigned char x_pos, y_pos, place, adjust;
    x_pos = y_pos = place = adjust = 0;

    while (x_pos < (string_length + 16)) { //loop for the entire scroll. Iterations = Rows + String_length
        Lcd_Set_Cursor(1, 16 - x_pos); //sets the cursor's start position
        y_pos = place; //sets the character to display first
        while (y_pos <= x_pos + adjust) { //Loads the required characters for a single iteration
            Lcd_Print_Char(string_select[y_pos], 1800); //Displays each character.
            y_pos++;
        }
        x_pos++;
        if (x_pos > 15 && place < (17 + string_length)) { //Checks if the cursor is at 1,1 and if 
            x_pos = 15; //Holds the cursor at 1,1.
            if (place == (16 + string_length)) {
                x_pos = 16 + string_length; //Used to stop the entire scroll operation when complete
            }
            place++;
            if (x_pos + adjust < string_length) { //
                adjust++; //increases the viewing range of characters
            }
        }
    }
}
//==============================================================================

//====================================HANDLES EEPROM FUNCTIONS.....................
void Write_eeprom(unsigned short add_ress, unsigned short da_ta){
 unsigned char INTCON_save = INTCON, PIE1_save = PIE1, PIR2_save = PIR2, PIE2_save = PIE2; //Backup Interrupt registers.
 //Start writing
 EEADR = add_ress;
 EEDATA = da_ta;  //EEPGD = 0; WREN = 1;   // clearing this bit grants access to data EEPROM. Also enable write cycle.
 
 EECON1bits.EEPGD = 0;//EECON1 = 4;
 EECON1bits.WREN = 1;
 
 INTCON = 0;
 PIE1 = 0;
 PIE2 = 0;
 EECON2 = 85;
 EECON2 = 170; //WR = 1; cleared by hardware
 EECON1bits.WR = 1;//EECON1 = 6;
 
 while(EECON1bits.WR);//__delay_ms(500);
 INTCON = INTCON_save; //Was previously on 208;
 PIE1 = PIE1_save; //Was previously on 32;
 //PIE2 = 16;   // ENABLE Interupts if you were using them previously.
 EECON1 = 0;
 //__delay_ms(100);
 PIR2 = PIR2_save;
}


unsigned short Read_eeprom(unsigned short add_ress){
   EEADR = add_ress; //EEPGD = 0; RD = 1;
   EECON1 = 1;
   __delay_ms(10);
return EEDATA;
}
//==============================================================================

//====================================HANDLES ON AND OFF STATES.....................
void activate_deactivate(unsigned char state_select){
    //DEACTIVATION
    if(state_select == 2){
        //Shutdown.
        relay_control = ENABLE;//Turn on relay to cut off supply to the building. pnp transistor used.
        display_permit = 2; //Change display message
        stop_clock = 0;
        relay_state = ACTIVE; //Relay activates to cut off supply.
        Write_eeprom(254, relay_state);//Saves relay status in RAM
        //Stop countdown;
        T1CON &= 254;
    }
    //ACTIVATION
    if(state_select == 4){
        //Restore.
        relay_control = DISABLE;//Turn off relay to supply power to the building.
        display_permit = 0; //Change display message
        stop_clock = 0;
        relay_state = INACTIVE; //Relay deactivates to restore supply.
        Write_eeprom(254, relay_state);//Saves relay status in RAM
        //Start countdown;
        T1CON |= 1;
        TMR1H = 11; //Starts timer again
        TMR1L = 219;//Timer 1 registers for timing configurations
    }
}
//==============================================================================

//====================================USART definitions.........................
void my_init_usart(unsigned int baud_rate){
     //This function also adds USART reception.

                   TXSTAbits.BRGH = 1; //High Speed, Low Speed select
                   baud_rate/=8;// divided by 8 to fit into 16bits
                   baud_rate = 62500/baud_rate;
                   X = baud_rate - 1;
                   SPBRG = X; //BR generator register
                   TXSTAbits.SYNC = 0;
                   RCSTAbits.SPEN = 1;
                   
                   //USART receive routine
                   PIE1bits.RCIE = 1; //USART interrupt enable bit.
                   RCSTAbits.CREN = 1; // Start receiving
                   __delay_ms(200);
                   gsm_config();
                   }

void my_usart_write(unsigned char byte){
     // Since the USART read function is already active, deactivate it and
     //reactivate after Transmission
                    TXSTAbits.BRGH = 1;
                    SPBRG = X; //BR generator register
                    TXSTAbits.SYNC = 0; //USART mode select bit
                    RCSTAbits.SPEN = 1;  //Serial port enable bit
                    TXREG = byte;
                    TXSTAbits.TXEN = 1;
                    while(TXSTAbits.TRMT==0); //waits for each byte to get sent.
                   }

void my_usart_write_text(unsigned char *text){
unsigned char y;
y = 0;
    while(text[y]!= '\0'){
        my_usart_write(text[y]);
        y++;    
    }
}
//==============================================================================

//====================================GSM Config definitions....................
void gsm_config(void){
//delay_ms(2000); //remove comment when its time to  program the MCU
my_usart_write_text(ATE0);   // AT command for Echo OFF
my_usart_write(13);
my_usart_write(10);
__delay_ms(50);
my_usart_write_text(AT_CMGF);
my_usart_write(13);
my_usart_write(10);
__delay_ms(50);
my_usart_write_text(AT_CNMI);
my_usart_write(13);
my_usart_write(10);
}
//==============================================================================

//====================================SENDS USART MESSAGES
void send_message(unsigned char *number, unsigned char message_select){
unsigned char cursor = 0;
my_usart_write(13);
my_usart_write(10);//REMOVE WHEN TESTING THE MAIN CIRCUIT   
my_usart_write_text(AT_CMGF);
my_usart_write(13);
my_usart_write(10);
__delay_ms(50);// change delay to 500ms when doing main testing
my_usart_write_text(AT_CMGS);
my_usart_write(34);
my_usart_write_text(number);
my_usart_write(34);
my_usart_write(13);
my_usart_write(10);
__delay_ms(50);// change delay to 500ms when doing main circuit testing
    switch (message_select) {
        case 82: my_usart_write_text(R_UNI);
            break;
        case 83: my_usart_write_text(RECHARGE);
                my_usart_write(32);
                my_usart_write_text(SUCC);
            break;
        case 84: my_usart_write_text(UNITS_LABEL);
                 my_usart_write(32);
                 my_usart_write_text(meter);
                 my_usart_write_text("KVAH.");
            break;
        case 85: my_usart_write_text(RECHARGE);
                my_usart_write(32);
                my_usart_write_text("UN");
                my_usart_write_text(SUCC);
            break;
        case 86: my_usart_write_text(CREDIT);
            break;
        case 87: my_usart_write_text(SUCC);
            break;
        case 88: my_usart_write_text(P_THEFT);
                 my_usart_write(13);
                 my_usart_write(10);
                 my_usart_write_text("TAMPER CODE: ");
                 my_usart_write_text(meter);
                 my_usart_write(13);
                 my_usart_write(10);
                 my_usart_write_text(DATE);
            break;
    }
my_usart_write(26);
my_usart_write(13); //New line is not needed in the real circuit
my_usart_write(10);
}
//==============================================================================

//====================================STRING FIXER into interrupt variables
void fix(void){
switch (index){
case 0: byte_message = CMT_notify[ql];
        second_byte_message = CCLK_notify[LM];
        word_size = 5;
        second_word_size = 6;
        break;

case 1: byte_message = 34;
        word_size = 1;
        break;

case 3: byte_message = LOAD_CMD[ql];
        second_byte_message = DEV_C[LM];
        third_byte_message = ADD_NUM[dd];
        forth_byte_message = BALANCE[fif];
        word_size = 5;
        second_word_size = 4;
        third_word_size = 7;
        forth_word_size = 8;
        break;
 /*      
case 5: byte_message = INVERTER[ql];
        second_byte_message = PHCN[dd];
        third_byte_message = AUTO[LM];
        forth_byte_message = GEN[dev];
        fifth_byte_message = OFF[fif];
        word_size = 9;
        second_word_size = 6;
        third_word_size = 5;
        forth_word_size = 4;
        fifth_word_size = 8;
        break;*/
        }}
//==============================================================================

//====================================MANUALLY STORE IN RAM.....................
unsigned char store_in_RAM(void){ //From USART to RAM
 unsigned char test;
 unsigned short set=0;  //change back to unsigned int if not working
  test = RCREG;
  if(test == 46){reg_count = exx;} //used to stop copying characters after '.' is detected in the SMS
  STATUSbits.IRP = 1;  //Bank select together with FSR
  FSR = 288 + exx; //RAM address is modified with date and time once received
  INDF = test; //RAM data
  STATUSbits.IRP = 0;
  if(test!=46){exx++;}
  if(exx == reg_count){ 
     if(index==2){ //PORTBbits.RB5
         if(second_word_size == 6){
           swift = 1;
           exx = 0;
           index = 0;
         }else index++;  
     }
     if(index==4){
        //set = 1;
        exx = 0;
        index = 0;   //monitor continuously
        if(CMD == 3){
           set = 3; 
           CMD = 0;
        }
     }
     }
  return set;
}
//==============================================================================

//====================================HANDLES RELOADING OF UNITS................
void reloading(void){
    unsigned char collect;
    unsigned short counter, me;
    unsigned int units;
    counter = me = units = 0;
    while(counter < 8) {//Check if the reload process code is accurate.
        STATUSbits.IRP = 1;
        FSR = 304 + counter; //RAM address
        collect = INDF;
        STATUSbits.IRP = 0;
        //my_usart_write(collect); //Was used to test
        if (collect == LOAD_PIN[counter] && counter < 6) {//Checking process code
            me++; 
        }
        counter++;
        if (me == 6 && counter >= 7) { //if process code is accurate
            //my_usart_write(collect);
            if (collect > 64 && collect < 71) { //converts hex letters from characters to decimal integer
                collect -= 55;
            }else if (collect > 47 & collect < 58) {//converts hex numbers from characters to decimal integer
                collect -= 48;
            }
            units += collect;//6+4. e.g 64 to integer is 100
            
            //Actual reload
            if(counter == 8){
                units -= collect;//10-4.
                units *= 16; //6*16. convert hex to integer.
                units += collect; //96+4 = 100.
                realUnits *= 21; //Does this to efficiently add the previous units.
                realUnits += units; //Stores in float to deal with decimal fractions
                realUnits /= 21;
                StoreDecimalFraction(realUnits);//Stores new units on ROM
                displayUnits = realUnits;
                //Alert the user through LCD
                Lcd_Clear();
                Lcd_Set_Cursor(1, 5);
                Lcd_Print_String(RECHARGE);
                Lcd_Set_Cursor(2, 4);
                Lcd_Print_String(SUCC);
                //Alert the user through SMS.
                //get phone number first.
                counter = 0;
                while (counter < 14) {
                    STATUSbits.IRP = 1;
                    FSR = 289 + counter; //RAM address
                    collect = INDF;
                    STATUSbits.IRP = 0;
                    phone_num[counter] = collect;
                    counter++;
                }
                send_message(phone_num, 'S'); //48
                __delay_ms(700);
                //display_permit = 0;
            }
        }
    }
    if(me < 6){ //If process code is wrong or anything else goes wrong 
            //Alert the user through LCD
            Lcd_Clear();
            Lcd_Set_Cursor(1, 5);//Avoids continuously loading these characters to the LCD. Might remove this
            Lcd_Print_String(RECHARGE);
            Lcd_Set_Cursor(2, 3);
            Lcd_Print_String("UN");
            Lcd_Set_Cursor(2, 5);
            Lcd_Print_String(SUCC);
            //Alert the user through SMS.
            //get phone number first.
            counter = 0;
            while (counter < 14){
                STATUSbits.IRP = 1;
                FSR = 289 + counter; //RAM address
                collect = INDF;
                STATUSbits.IRP = 0;
                phone_num[counter] = collect;
                counter++;
            }
            send_message(phone_num, 'U'); //48
            __delay_ms(700);
            display_permit = 2;
        }
}
//==============================================================================

//====================================HANDLES ANALOGUE SIGNAL MEASUREMENTS......
void my_ADC_Init(void){
 TRISA |= 3; //Sets  RA0 and RA1 as input
 ANSEL |= 3; //Sets RA0 and RA1 as analogue pins.
 //ANSELH = 0;
 ADCON0 = 128; //(Fosc/32)
 ADCON1 = 128; //sets ADFM = 1   
}

unsigned int my_ADC_Read(unsigned char channel) {
    unsigned int digi_value;
    ADCON0 &= 195; //Clears previous channel selection
    ADCON0 |= channel << 2; //set channel
    ADCON0bits.ADON = 1; //ON ADC
    __delay_us(20); //Acquisition time
    ADCON0bits.GO_DONE = 1; // start converSion
    while (ADCON0bits.GO_DONE); //polling GO/DONE bit
    digi_value = ADRESL;
    digi_value += ADRESH * 256; //moves ADRESX bits to integer
    //ADCON1 = 135; //was set back to all digital becos of the way i used portA pins
    return digi_value;
}
//==============================================================================

//====================================HANDLES ANALOGUE VARIABLE CALCULATIONS
void load_monitoring(void) {
    unsigned short plus; //denominator, minus;
    float decimal, volt_s, amp_s = 0;

    //Voltage measurement from voltage sensor
    decimal = my_ADC_Read(0); // Get 10-bit results of AD conversion
    decimal *= 0.0048875855; //count to value conversion
    //decimal-=0.1309844824;  // Diode rectifier Inverse voltage drop (Subtraction due to cancel parasitic capacitance)
    decimal *= 1.0367884886; //1.0755782848; // instrument error correction.
    decimal *= 0.7071067811; // Vmax to Vrms conversion multiple
    decimal *= 0.95029934; // Practical calibration factor
    decimal *= 75.0740740740; //61.6060606060; // AC resistor inverse voltage drop (RB/RA+RB+RC).
    //decimal *= 1.0091743119;//1.0068807339; // instrument error correction. Returns 220V
    volt_s = decimal;

    //Current measurement from current sensor
    plus = 0; //This variable was reused here.
    while (plus < 770){
        decimal = my_ADC_Read(1);
        decimal *= 0.0048875855; //count to value conversion
        if (decimal > amp_s){
            amp_s = decimal;
        }
        plus++;
    }
    if (amp_s > 2.429){
        amp_s -= 2.429;//Was on 2.5 + 0.266; //Remove offset
    }else amp_s = 0;
    amp_s *= 12; //Was on 12 //Current conversion
    amp_s *= 0.7071067811; // Vmax to Vrms conversion multiple
    
    /*
    //Power Factor measurement
    decimal = time_difference * 0.128; //TRM0 count to time value conversion
    decimal *= 0.000000004846; //time to radians conversion for 60Hz, in preparation for cosine.
    square = 1;          // 377.076/1000(mAmps to Amps)
    cos_ans = 1;
    minus = 0;
    plus = 0;
    while (minus < 2) { //cosine
        square *= decimal; //square *= decimal*decimal;
        square *= decimal;
        denominator = factorial[minus + plus];
        cos_ans -= square / denominator;
        minus++;
        while (plus < 2) {
            square *= decimal; //square *= decimal*decimal;
            square *= decimal;
            denominator = factorial[minus + plus];
            cos_ans += square / denominator;
            plus++;
        }
    }*/
    power = (int) volt_s * amp_s; //remember to round to the nearest dp
    //power = amp_s * 1000;
}
//==============================================================================

//====================================HANDLES INTERGER TO STRING CONVERTION=====
void my_inttoStr(unsigned int integer){ //The function handles five characters max
char va;
unsigned int y;
unsigned int count;
int div;
int f_ans;
int sub;
float ans;

va=0;
while(va<=3){   //erasing value array before use.
      meter[va] = 0;
      va++;
      }
div=10000;
sub = 0;
y=5;
while(div>integer){
      div/=10;
      y-=1;
      }
for(count=1;count<=y;count++){
    ans = integer/div;
    f_ans = (int)ans;
    va = f_ans - sub;
    div/=10;
    va+= 48;
    meter[count-1] = va;
    sub = f_ans*10;
    }
}
//==============================================================================

//====================================HANDLES DECIMAL FRACTION STORAGE==========
void StoreDecimalFraction(float decimal_fraction){
    char carry;
    int aa, bb, cc;
    
    aa = cc = decimal_fraction;
    decimal_fraction *= 1000;
    cc *= 1000;
    bb = decimal_fraction - cc;
    //Save whole number part
    carry = 0;
    while(aa >= 256){
        aa -= 256;
        carry++;
    }
    Write_eeprom(39, carry);
    Write_eeprom(40, aa);
    //Save decimal number part
    carry = 0;
    while(bb >= 256){
        bb -= 256;
        carry++;
    }
    Write_eeprom(41, carry);
    Write_eeprom(42, bb);
}

float ReadDecimalFraction(void){
    float fraction;
    unsigned int whole, dec_frac;
    //Reads whole number part
    whole = Read_eeprom(39);
    whole *= 256;
    whole += Read_eeprom(40);
    //Reads decimal number part
    dec_frac = Read_eeprom(41);
    dec_frac *= 256;
    dec_frac += Read_eeprom(42);
    //Combine both 
    fraction = dec_frac;
    fraction /= 1000;
    fraction += whole;
    return fraction;
}
//==============================================================================

//====================================HANDLES TAMPER DETECTION==================
void tamper_routine(void){
unsigned int tamper_code; //Consider converting tamper_code to local variable instead
    if (TAMPER == 0 && sample == MUTE){ //Begin triggering the transistor 5 times
        sample = 1;
    }
    
    if (sample != MUTE){
        if (sample == 6){ //6-1 = 5
            tamper_code = Read_eeprom(20); //Get the past assert state from ROM, which is the tamper state
            if (tamper_code == 0){
                //don't send message
            }
            tamper_code += Read_eeprom(22); //Adds the past error code the resulting error code
            Write_eeprom(20, TAMPER);//Store Tamper state in ROM, in char format. This will be the future assert state state
            Write_eeprom(22, tamper_code); //Update to the current error code.
            //Prepare message
            my_inttoStr(tamper_code);
            
            //Get date and time
            swift = 0;
            my_usart_write_text(AT_CCLK); //Requests real time form GSM module
            my_usart_write(13); //Remove this when loading finally 
            my_usart_write(10); //into PIC.
            while(!swift); //Wait until the request has been granted by the GSM module
            swift = 0;
            while(swift < 15){
                STATUSbits.IRP = 1;
                FSR = 289 + swift; //RAM address
                DATE[swift] = INDF; //adds full stop to the end of log data in RAM.
                STATUSbits.IRP = 0;
                swift++;
            }
            
            //Get recipient
            swift = 0;
            while (swift < 14){ //Acquire saved phone number. It should be the service providers number
                phone_num[swift] = Read_eeprom(swift + 1);
                //phone_no[swift] = parse;
                swift++;
            }
            
            //Send anti-theft report
            send_message(phone_num, 'X');
            sample = MUTE - 1;
        }
        sample++;
    }
    ASSERT = 1;//Assert
    __delay_ms(100);//time constant of the RC circuit.
    ASSERT = 0;
}
//====================================================================

//Prepaid .h
//========================FUNCTION declarations.....................
void Lcd_home_screen(void);
void Lcd_SetBit(char data_bit);
void Lcd_Cmd(char a);
void Lcd_Clear(void);
void Lcd_Set_Cursor(char u, char b);
void Lcd_Start(void);
void Lcd_Print_Char(char da_ta, int delay_print);
void Lcd_Print_String(char *a);
void Lcd_home_screen();
void L_R_TextScroll(unsigned char *string_select, unsigned short string_length);
void Write_eeprom(unsigned short add_ress, unsigned short da_ta);
unsigned short Read_eeprom(unsigned short add_ress);
void activate_deactivate(unsigned char state_select);
void my_init_usart(unsigned int baud_rate);
void my_usart_write(unsigned char byte);
void my_usart_write_text(unsigned char *text);
void gsm_config(void);
void send_message(unsigned char *number, unsigned char message_select);
void fix(void);
unsigned char store_in_RAM(void);
void reloading(void);
void my_ADC_Init(void);
unsigned int my_ADC_Read(unsigned char channel);
void load_monitoring(void);
void my_inttoStr(unsigned int integer);
void StoreDecimalFraction(float decimal_fraction);
float ReadDecimalFraction(void);
void tamper_routine(void);
//==============================================================================

//========================STRING CONSTANTS definitions..........................
unsigned char meter[5];
unsigned char phone_num[15];
unsigned char DATE[15];
unsigned const int factorial[] = {2,24,720,40320}; 
unsigned const char ATE0[] = "ATE0";
unsigned const char AT_CMGF[] = "AT+CMGF=1";
unsigned const char AT_CNMI[] = "AT+CNMI=2,2,0,0,0";
unsigned const char AT_CMGS[] ="AT+CMGS="; 
unsigned const char AT_CCLK[] ="AT+CCLK?";
unsigned const char CCLK_notify[] = "+CCLK:";
unsigned const char CMT_notify[] = "+CMT:";
unsigned const char UNITS_LABEL[] = "UNITS:";
unsigned const char LOAD_LABEL[] = "LOAD(W):";
unsigned const char R_UNI[] = "RELOAD UNITS!!!";
unsigned const char LOAD_CMD[] = "LOAD,";  
unsigned const char LOAD_PIN[] = "194CDBXX";
unsigned const char RECHARGE[] = "RECHARGE";
unsigned const char SUCC[] = "SUCCESSFUL";
unsigned const char ADD_NUM[] = "ADDNUM.";
unsigned const char P_THEFT[] = "POWER THEFT DETECTED!";
unsigned const char BALANCE[] = "BALANCE.";
unsigned const char ERASE[] = "      ";
//==============================================================================

//========================INTERFILE variable definitions (externs)..............
unsigned char byte_message;
unsigned char second_byte_message;
unsigned char forth_byte_message;
unsigned char third_byte_message;
unsigned char fifth_byte_message;
unsigned char display_permit;
unsigned char garbage;
unsigned char swift;
unsigned short time_difference;
unsigned short relay_state;
unsigned short stop_clock;
unsigned short CMD;
unsigned short X;
unsigned short sample;
unsigned int displayUnits;
unsigned int ql, fif;
unsigned int LM, dd;
unsigned int index;
unsigned int word_size;
unsigned int second_word_size; 
unsigned int third_word_size;
unsigned int forth_word_size;
unsigned int fifth_word_size;
unsigned int reg_count;
unsigned int exx;
unsigned int power;
float realUnits;
//==============================================================================

//========================NUMERIC ARRAY definitions.............................

//==============================================================================

//========================INTEGER CONSTANTS definitions.............................
#define ENABLE 1
#define DISABLE 0
#define ACTIVE 8
#define INACTIVE 3
#define MUTE 9
#define ASSERT PORTAbits.RA4
#define TAMPER PORTAbits.RA5
//==============================================================================
COMMANDS.
SMS UNITS Reload FORMAT: LOAD,Recharge-code.
eg: LOAD,194CDB64.

SMS SAVE NUMBER ACTION FORMAT: ADDNUM.
eg: ADDNUM.

SMS BALANCE CHECK ACTION FORMAT: BALANCE.
eg: BALANCE.

ANTI THEFT REPORT FORMAT:
POWER THEFT DETECTED!
CODE: Device code - Alert code eg: 523-323
Type: TAMPER / BYPASS
DATE: 01/01/2020.
