# stm32-HAL-HCSR04Test
stm32利用HCSR04进行超声波测距
## 硬件原理
![电路接线图](https://github.com/wahtcanisay/stm32-HAL-HCSR04Test/blob/main/%E6%8E%A5%E7%BA%BF%E5%9B%BE.png)
电路接线图  
传感器Trig引脚接单片机输出，设置为推挽模式；该引脚受到脉冲信号(大于10us)即发送超声波(发送要消耗约0.2ms)  
Echo引脚接定时器CH1，CH1选择上升沿脉冲直接模式；也接入CH2，CH2选择下降沿脉冲间接模式  
超声波发送完成，Echo会产生上升沿，接收返回的超声波之后，Echo引脚会产生下降沿  
当通道CH1捕获到上升沿，会将标志位CC1拉高并CNT值保存到CCR1当中；CH2捕获到下降沿，会将标志位CC2拉高并CNT值保存到CCR2当中，查询标志位即可知道当前通道有没有捕捉到信号变化  
## 软件实现
### cubemx部分
首先在`SYS`中选择`Serial Wire`  
将`PA0`选为输出，初始电压低电压，推挽模式，低速  
选择`TIM1`，`CH1`选中`Input Caputure direct mode`(输入捕获直接)，`CH2`选中`Input Capture indirect mode`(输入捕获间接)  
`Clock Source`选择`Internal Clock`,PSC设置为7(分辨率为1us),ARR设为65535(65.536ms>38ms最大脉宽长度)，上计数，使能ARR预加载  
在参数设置中找到`Input Caputure Channel 1`，设置为`Rising Edge`，`CH2`对应选择`Falling Edge`  
不设置分频系数,设为`No division`,`Input Filiter`也不设置，写为0   
PC13设置为输出开漏模式，初始高电压，低速  
### keil5部分
**新接口：**
```c
__HAL_TIM_CLEAR_FLAG(__HANDLE__, _CCx_)
__HAL_TIM_GET_FLAG(__HANDLE__, _CCx_)
```
作用：清除/查询ccx标志位的值   
```c
HAL_TIM_IC_Start(HANDLE, CHANEL)

HAL_TIM_IC_Stop(HANDLE, CHANEL)
```
作用：启动/关闭输入捕获，通过控制时基单元开关和输入捕获对应通道开关实现
```c
int main(){

    while(1){
        
        //#1.清除CNT值
        __HAL_TIM_SET_COUNTER(&htim1, 0);

        //#2.清除CC1、CC2标志位
        __HAL_TIM_CLEAR_FLAG(&htim1, TIM_FLAG_CC1);
        __HAL_TIM_CLEAR_FLAG(&htim1, TIM_FLAG_CC2);

        //#3.启动定时器
        HAL_TIM_IC_Start(&htim1, TIM_CHANNEL_1);
        HAL_TIM_IC_Start(&htim1, TIM_CHANNEL_2);

        //#4.向Trig引脚发送脉冲
        HAL_GPIO_WritePin(GPIOA, GPIO_PIN_0, GPIO_PIN_SET);//向PA0写1
        for(uint32_t i = 0;i < 10, i++); //延迟确保发送时间大于10us(一次for循环消耗约 8指令周期*1/8MHz≈1us)
        HAL_GPIO_WritePin(GPIOA, GPIO_PIN_0, GPIO_PIN_RESET);//向PA0写0

        //#5.等待测量结束
        uint8_t success = 0;//测量是否成功，0-失败，1-成功
        uint32_t expireTime = HAL_GetTick() + 50; //设置最长的等待时间
        while(expireTime > HAL_GetTick()){//判断是否超时
            //获取标志位
            uint32_t cc1Flag = __HAL_TIM_GET_FLAG(&htim1, TIM_FLAG_CC1);
            uint32_t cc2Flag = __HAL_TIM_GET_FLAG(&htim1, TIM_FLAG_CC2);
            if(cc1Flag && cc2Flag){
                //cc1 cc2均为1
                success = 1;//测量成功
                break;//跳出循环   
            }
        }

        //#6.关闭定时器
        HAL_TIM_IC_Stop(&htim1, TIM_CHANNEL_1);
        HAL_TIM_IC_Stop(&htim1, TIM_CHANNEL_2);

        //#7.计算测量结果
        if(success){
            //测量已经成功,读取ccx值
            uint16_t ccr1 = __HAL_GET_COMPARE(&htim1, TIM_CHANNEL_1);
            uint16_t ccr2 = __HAL_GET_COMPARE(&htim1, TIM_CHANNEL_2);
            float pulseWidth = (ccr2 - ccr1) * 1e-6f;//计算脉宽
            float distance = 340.0f * pulseWidth / 2.0f;//计算距离
            if (distance < 0.2f){
                //点亮板载LED
                HAL_GPIO_WritePin(GPIOC, GPIO_PIN_13, GPIO_PIN_RESET);
            }
            else{
                //否则熄灭LED
                HAL_GPIO_WritePin(GPIOC, GPIO_PIN_13, GPIO_PIN_SET);
            }
        }
    }
}
```