## 进度报告第二期

### ATK-ESP8266使用

>原理：基于串口通讯+AT指令（通过向串口发送AT相关指令实现wifi连接）
>
>代码： 使用了原子哥的ATK-ESP8266 的库，atk连接wifi的代码是根据原子给的源码进行参考，书写完成的。
>
>```c
>//主函数
>/*
>esp8266连接wifi是通过AT指令完成的，这部分应该是封装在atk-esp8266里面
>我们要连接wifi只需要通过串口通讯发送相应指令给esp8266，实现相应功能
>*/
>#include "common.h"//原子哥的库
>#include "stdlib.h"
>u8 wifi_sta_connect()
>{
>  u8 *p;
>
>  atk_8266_send_cmd("AT+CWMODE=1","OK",50);//设置了sta模式
>  atk_8266_send_cmd("AT+RST","OK",20);//重启生效
>	delay_ms(1000);
>	delay_ms(1000);
>	delay_ms(1000);
>	printf("AT RST success,wifi is connecting...");
>  atk_8266_send_cmd("AT+CWJAP=wifista_ssid,wifista_password","WIFI GOT IP",300);//连接热点
>  atk_8266_send_cmd("AT+CIPMUX=0","OK",20)//设置为单连接模式
>  if(atk_8266_send_cmd("AT+CIPSTATUS?","OK",50))//检查连接是否成功
>	  printf("wifi has connected");
> //tcp连接部分，目前在研究中	/*while(atk_8266_send_cmd("AT+CIPSTART=\"TCP\",\"xxx.xxx.xxx.xxx\",8086","OK",200));
>	atk_8266_send_cmd("AT+CIPMODE=1","OK",200);
>   */
>    
>}
>
>```
>
>接下来为原子哥的库：从源代码上搬运过来（原子哥的源码多加了许多奇怪的东西）
>
>### 串口3函数
>
>```c
>//串口通讯设置：使用了串口3
>//由于基本配置跟例程的串口通讯配置差不多不列出配置代买
>//以下为发送给esp8266串口的串口发送代码
>void u3_printf(char* fmt,...) //传入了指令或者数据 
>{  
>	u16 i,j; 
>	va_list ap; //创建一个va_list结构体空间
>	va_start(ap,fmt);//将ap中指针指向fmt数据，得到fmt开头地址
>	vsprintf((char*)USART3_TX_BUF,fmt,ap);//将可变参数列表存入要发送的数据中
>	va_end(ap);//清空va_list可变参数列表：
>	i=strlen((const char*)USART3_TX_BUF);		//量取数据长度
>	for(j=0;j<i;j++)							//发送数据
>	{
>	  while(USART_GetFlagStatus(USART3,USART_FLAG_TC)==RESET); //检验数据是否发送成功
>                                                        //USART_FLAG_TC为发送完成标志位
>		USART_SendData(USART3,USART3_TX_BUF[j]);//发送数据 
>	} 
>}
>
>
>
>```
>
>### common库函数
>
>```C
>//common函数用于连接wifi的底层函数
>//以下重点介绍几个函数其余不作注解，这一段可以直接跳过
>#include "common.h"
>
>
>const u8* portnum="8086";	
>//ÉèÖÃ×Ô¼ºµÄÈÈµã
>const u8* wifista_ssid="HUWEI";			//
>const u8* wifista_encryption="wpawpa2_aes";	//
>const u8* wifista_password="hhp073156"; //
>
>void atk_8266_at_response(u8 mode)
>{
>	if(USART3_RX_STA&0X8000)		//
>	{ 
>		USART3_RX_BUF[USART3_RX_STA&0X7FFF]=0;//
>		printf("%s",USART3_RX_BUF);	//
>		if(mode)USART3_RX_STA=0;
>	} 
>}
>
>u8* atk_8266_check_cmd(u8 *str)
>{
>	
>	char *strx=0;
>	if(USART3_RX_STA&0X8000)		//
>	{ 
>		USART3_RX_BUF[USART3_RX_STA&0X7FFF]=0;//
>		strx=strstr((const char*)USART3_RX_BUF,(const char*)str);
>	} 
>	return (u8*)strx;
>}
>
>
>//·¢ËÍÖ¸Áî
>u8 atk_8266_send_cmd(u8 *cmd,u8 *ack,u16 waittime)
>{
>	u8 res=0; 
>	USART3_RX_STA=0;
>	u3_printf("%s\r\n",cmd);	//
>	if(ack&&waittime)		//
>	{
>		while(--waittime)	//
>		{
>			delay_ms(10);
>			if(USART3_RX_STA&0X8000)//
>			{
>				if(atk_8266_check_cmd(ack))
>				{
>					printf("ack:%s\r\n",(u8*)ack);
>					break;//
>				}
>					USART3_RX_STA=0;
>			} 
>		}
>		if(waittime==0)res=1; 
>	}
>	return res;
>} 
>
>u8 atk_8266_send_data(u8 *data,u8 *ack,u16 waittime)//发送数据函数，同以上命令函数
>{
>	u8 res=0; 
>	USART3_RX_STA=0;
>	u3_printf("%s",data);	//
>	if(ack&&waittime)		//
>	{
>		while(--waittime)	//
>		{
>			delay_ms(10);
>			if(USART3_RX_STA&0X8000)//
>			{
>				if(atk_8266_check_cmd(  ack))break;// 
>				USART3_RX_STA=0;
>			} 
>		}
>		if(waittime==0)res=1; 
>	}
>	return res;
>}
>
>u8 atk_8266_quit_trans(void)//退出透传
>{
>	while((USART3->SR&0X40)==0);	//
>	USART3->DR='+';      
>	delay_ms(15);					//´
>	while((USART3->SR&0X40)==0);	//
>	USART3->DR='+';      
>	delay_ms(15);					//´
>	while((USART3->SR&0X40)==0);	//
>	USART3->DR='+';      
>	delay_ms(500);					//
>	return atk_8266_send_cmd("AT","OK",20);//
>}
>
>u8 atk_8266_apsta_check(void)//退出透传
>{
>	if(atk_8266_quit_trans())return 0;			// 
>	atk_8266_send_cmd("AT+CIPSTATUS",":",50);	//
>	if(atk_8266_check_cmd("+CIPSTATUS:0")&&
>		 atk_8266_check_cmd("+CIPSTATUS:1")&&
>		 atk_8266_check_cmd("+CIPSTATUS:2")&&
>		 atk_8266_check_cmd("+CIPSTATUS:4"))
>		return 0;
>	else return 1;
>}
>
>u8 atk_8266_consta_check(void)//关闭透传
>{
>	u8 *p;
>	u8 res;
>	if(atk_8266_quit_trans())return 0;			// 
>	atk_8266_send_cmd("AT+CIPSTATUS",":",50);	//
>	p=atk_8266_check_cmd("+CIPSTATUS:"); 
>	res=*p;									//	
>	return res;
>}
>
>void atk_8266_get_wanip(u8* ipbuf)//得到ip地址
>{
>	u8 *p,*p1;
>		if(atk_8266_send_cmd("AT+CIFSR","OK",50))//
>		{
>			ipbuf[0]=0;
>			return;
>		}		
>		p=atk_8266_check_cmd("\"");
>		p1=(u8*)strstr((const char*)(p+1),"\"");
>		*p1=0;
>		sprintf((char*)ipbuf,"%s",p+1);	
>}
>
>```
>
>### common库中的重点函数
>
>```c
>//第一个要重点介绍的函数
>//数据发送
>u8 atk_8266_send_cmd(u8 *cmd,u8 *ack,u16 waittime)//传入的参数有指令cmd，应答ack（固定好的），等待时间：waittime，需要传入正确指令以及对应答才能正确发送
>{
>	u8 res=0; //标志
>	USART3_RX_STA=0;
>	u3_printf("%s\r\n",cmd);	//发送指令给ATK-ESP8266
>	if(ack&&waittime)		//反馈给串口应答，判断是否有应答以及等待时间
>	{
>		while(--waittime)	//
>		{
>			delay_ms(10);
>			if(USART3_RX_STA&0X8000)//接收数据的中断位，判断最高位是否为1，是1则开启接收
>			{
>				if(atk_8266_check_cmd(ack))//返回应答
>				{
>					printf("ack:%s\r\n",(u8*)ack);//输出到串口界面
>					break;//
>				}
>					USART3_RX_STA=0;//完成接收
>			} 
>		}
>		if(waittime==0)res=1; 
>	}
>	return res;
>} 
>
>u8* atk_8266_check_cmd(u8 *str)//传入的ack
>{
>	
>	char *strx=0;
>	if(USART3_RX_STA&0X8000)		//接收判断
>	{ 
>		USART3_RX_BUF[USART3_RX_STA&0X7FFF]=0;//标记结束标志为0
>        //接收到数据中有指令以及应答，strstr函数是从一大段字符串中找到字符的第一次出现的位置。
>		strx=strstr((const char*)USART3_RX_BUF,(const char*)str);//从接收数据中找到ack的位置。
>	} 
>	return (u8*)strx;//返回位置
>}
>```
>
>目前只用到以上函数，另外函数之后用到再详细解释。
>
>附上成功连接图
>
>![image-20200830181130771](C:\Users\86189\AppData\Roaming\Typora\typora-user-images\image-20200830181130771.png)、
>
>![image-20200830181220878](C:\Users\86189\AppData\Roaming\Typora\typora-user-images\image-20200830181220878.png)
>
>``` c
>
>```
>
>
>
>

