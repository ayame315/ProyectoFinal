                Proyecto Final: Conexión por forwarding entre SIM 800 y FRDM KL25Z para toma de datos por GPS 



Yeison Franco

Sebastian Vasquez

Yaneth Mejía


------------------------------------------------------------


Código:


#include "mbed.h"   
#include "stdio.h"   
#include "string.h"   

Timer t;    
Serial sim800(PTE0,PTE1,9600);      //Serial SIM800C     
Serial gps(PTE22,PTE23,9600);       //Serial NEO6MV2    
Serial pc(USBTX,USBRX,9600);        //Serial al computador     

char gprsBuffer[20];
char cDataBuffer[550];
char aux[100];
char resp[15];
int i=0;
int z=0;
int count=0;


/////////////FUNCIONES/////////////////////////////////////////////////////////

//Leer Bufer GPRS
int readBuffer(char *buffer,int count){
    int i=0; 
    t.start();  // Inicia timer
    while(1) {
        while (sim800.readable()) {
            char c = sim800.getc();
            if (c == '\r' || c == '\n') c = '$';
            buffer[i++] = c;
            if(i > count)break;
        }
        if(i > count)break;
        if(t.read() > 3) {
            t.stop();
            t.reset();
            break;
        }
    }
    wait(0.5);
    while(sim800.readable()){  // display the other thing...
        char c = sim800.getc();
    }
    return 0;
}

//Borrar Buffer GPRS
void cleanBuffer(char *buffer, int count){
    for(int i=0; i < count; i++) {
        buffer[i] = '\0';
    }
}

//Enviar Comando AT a SIM800
void sendCmd(char *cmd){
    sim800.puts(cmd);
}

//Esperar Respuesta de SIM800
int waitForResp(char *resp, int timeout){
    int len = strlen(resp);
    int sum=0;
    t.start();
    i = 0;
    while(1) {
        if(sim800.readable()) {
            char c = sim800.getc();
            //pc.printf("\nc:%c",c);
            aux[i] = c;
            i++;
            sum = (c==resp[sum]) ? sum+1 : 0;
            if(sum == len)break;
        }
        if(t.read() > timeout) {
            t.stop();
            t.reset();
            return -1;
        }
    }
    t.stop();                 // detiene timer  antes de retornar
    t.reset();                    // clear timer
    while(sim800.readable()) {      // display the other thing..
        char c = sim800.getc();
    }
    return 0;
}

//Enviar Comando y Esperar por Respuesta
int sendCmdAndWaitForResp(char *cmd, char *resp, int timeout){
    sendCmd(cmd);
    return waitForResp(resp,timeout);
}
        
//Rutina de inicio
int init_gprs(void){
        
    if (0 != sendCmdAndWaitForResp("AT\r\n", "OK", 3)){
        pc.printf("1");
        return -1;
    }
    
    if (0 != sendCmdAndWaitForResp("ATE0\r\n", "OK", 3)){
       pc.printf("2");
        return -1;
    }
    
    if (0 != sendCmdAndWaitForResp("AT+CGATT=1\r\n", "OK", 3)){//inicia conexion GPRS
        pc.printf("3");
        return -1;
    }
    
    if (0 != sendCmdAndWaitForResp("AT+CSTT=\"em\"\r\n", "OK", 3)){
        pc.printf("4");
        return -1;
    }
    if (0 != sendCmdAndWaitForResp("AT+CIICR\r\n", "OK", 3)){
        pc.printf("5");
        return -1;
    }
    
    cleanBuffer(gprsBuffer,25);
    sendCmd("AT+CIFSR\r\n");
    wait(3);    
    pc.printf("6");
    
    
    return 0;
}

//Rutina de Desconexion
int end_gprs(void){
    if (0 != sendCmdAndWaitForResp("AT+CIPSHUT\r\n", "OK", 3)){
        return -1;
    }
     if (0 != sendCmdAndWaitForResp("AT+CGATT=0\r\n", "SHUT OK", 3)){  
        return -1;
    }
    return 0;
  }

//Datos GPS
void GPSinfo(char *cmd)
{
    char ns, ew, status;
    int date;
    float latitude, longitude, timefix, speed;
    int time, hora, minutos, segundos, fecha, dia, mes, anho;
    
    // Geographic position, Latitude and Longitude
    if(strncmp(cmd,"$GPRMC", 6) == 0) 
    {
        sscanf(cmd, "$GPRMC,%f,%c,%f,%c,%f,%c,%f,,%d", &timefix, &status, &latitude, &ns, &longitude, &ew, &speed, &date);
        time = timefix;     //Se convierte el valor a entero
        hora = (time/10000)-5;
        segundos = time%100;
        minutos = (time%10000)/100;
        fecha = date;
        dia = fecha/10000;
        anho = fecha%100;
        mes = (fecha%10000)/100;
        sim800.printf("Hora: %d:%d:%d, Latitud: %f %c, Longitud: %f %c, Speed: %.2f, Date: %d/%d/%d\n", hora, minutos, segundos, latitude/100, ns, longitude/100, ew, speed, dia, mes, anho);
        pc.printf("Hora: %d:%d:%d, Latitud: %f %c, Longitud: %f %c, Speed: %.2f, Date: %d/%d/%d\n", hora, minutos, segundos, latitude/100, ns, longitude/100, ew, speed, dia, mes, anho);
    }
}

//Enviar posicion
void posicion()
{
    char c;
    if(gps.readable())
    { 
        if(gps.getc() == '$');           // espera a $
        {
            for(int i=0; i<sizeof(cDataBuffer); i++)
            {
                c = gps.getc();
                if( c == '\r' )
                {
                    //pc.printf("%s\n", cDataBuffer);
                    GPSinfo(cDataBuffer);
                    i = sizeof(cDataBuffer);
                }
                else
                {
                    cDataBuffer[i] = c;
                }                 
            }
        }
    } 
}

/////////////FIN FUNCIONES/////////////////////////////////////////////////////


//PROGRAMA PRINCIPAL
int main(){

    inicio:
    if(init_gprs()<0){
        pc.printf("\n\niniciando\n\n");
        goto inicio;                      //En caso de no cargar correctamente el inicio
    }
    
    while(1){
        //sim800.printf("AT+CIPSTART=\"TCP\",\"179.13.40.24\",\"80\"\r\n");
        //pc.printf("AT+CIPSTART=\"TCP\",\"179.13.40.24\",\"80\"\r\n");
        Cliente:
        if (0 != sendCmdAndWaitForResp("AT+CIPSTART=\"TCP\",\"179.13.40.24\",\"80\"\r\n","OK", 5)){
            wait(3);
            pc.printf("\n\nConectando como cliente\n\n");
            goto Cliente;
        }
        
        Envio:
        cleanBuffer(gprsBuffer,10);
        if(0 !=sendCmdAndWaitForResp("AT+CIPSEND\r\n","",5)){  //devuelve control+Z
        wait(1);
        pc.printf("\n\nEnviando\n\n");
        goto Envio;
        }
        
        for(i=0;i<100;i++)
        {
            pc.printf("%c",aux[i]);
        }  
        wait(5);
        
        posicion();
        wait(5);
        
        cleanBuffer(aux,100);
        //GSM.putc((char)0x1A); 
        sim800.printf("\r\n");     //CTRL-Z
                        
        for(i=0;i<100;i++)
        {
            pc.printf("%c",aux[i]);
        }
        
        wait(1);
        goto Envio;     
    }
}
