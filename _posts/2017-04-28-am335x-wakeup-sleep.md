---
layout: post
title: "AM335x WakeUp&Sleep "
modified:
categories: blog
excerpt:
tags: [sleep wakeup]
image:
  feature:
---

最近在调试AM335X的休眠唤醒功能，可算是把我折腾惨了．

我们的需求是外部点火信号即ACC能唤醒睡眠中的系统，我们调试的时候用的是GPIO按键模拟ACC的方式．

### 第一个坑

首先根据AM335X的RM-25.2-Integration介绍,只有BANK0的GPIO才能配置成唤醒源.

> With four GPIO modules, the device allows for a maximum of 128 GPIO pins. (The exact number available varies as a function of the device configuration and pin muxing.) GPIO0 is in the Wakeup domain and may be used to wakeup the device via external sources. GPIO[1:3] are located in the peripheral domain.

即GPIO0可以作为Wakeup domain.而GPIO[1:3]则是peripheral domain.

在调试的时候,查看唤醒源.

```shell
cat /sys/kernel/debug/wakeup_sources
name            active_count    event_count     hit_count       active_since         
gpio-keys       0               0               0              0                    
am33xx-rtc      0               0    			0				0			
```

可以看见gpio-keys和rtc是配置成了唤醒源的,而且当按下按键的时候,系统是能够检测到的,说明GPIO相关的配置是没有问题的.

### 第二个坑

由于调了两天,始终毫无进展,所以我们换了上一版本的底板,发现居然能正常唤醒.经过对比,开始以为或许是因为新板子采用了特殊的电源管理,休眠的时候对GPIO有影响,最后发现不是,呵呵.最终"元凶"是4G模块,当系统进入休眠后,4G模块任然占有USB总线.最后在休眠之前先把4G模块电关掉,就能正常睡眠和唤醒了,呵呵!

电源管理这部分有空在写篇博客好好分析下.不过可以先看看AM335X休眠的一个关键函数.

```c

static int am33xx_pm_suspend(void)
{
	int state, ret = 0;

	struct omap_hwmod *gpmc_oh, *usb_oh, *gpio1_oh, *rtc_oh;

	usb_oh		= omap_hwmod_lookup("usb_otg_hs");
	gpmc_oh		= omap_hwmod_lookup("gpmc");
	gpio1_oh	= omap_hwmod_lookup("gpio1");	/* WKUP domain GPIO */
	rtc_oh		= omap_hwmod_lookup("rtc");

	omap_hwmod_enable(usb_oh);
	omap_hwmod_enable(gpmc_oh);
	
	/*
	 * Keep USB module enabled during standby
	 * to enable USB remote wakeup
	 * Note: This will result in hard-coding USB state
	 * during standby
	 */

	if (suspend_state != PM_SUSPEND_STANDBY)
	   omap_hwmod_idle(usb_oh);

	omap_hwmod_idle(gpmc_oh);

	/*
	 * Keep RTC module enabled during standby
	 * for PG2.x to enable wakeup from RTC.
	 */
	if ((omap_rev() >= AM335X_REV_ES2_0) &&
		(suspend_state == PM_SUSPEND_STANDBY)){
		omap_hwmod_enable(rtc_oh);
	}

	/*
	 * Disable the GPIO module. This ensure that
	 * only sWAKEUP interrupts to Cortex-M3 get generated
	 *
	 * XXX: EVM_SK uses a GPIO0 pin for VTP control
	 * in suspend and hence we can't do this for EVM_SK
	 * alone. The side-effect of this is that GPIO wakeup
	 * might have issues. Refer to commit 672639b for the
	 * details
	 */
	/*
	 * Keep GPIO0 module enabled during standby to
	 * support wakeup via GPIO0 keys.
	 */
	if ((suspend_cfg_param_list[EVM_ID] != EVM_SK) && (suspend_state != PM_SUSPEND_STANDBY)){
		omap_hwmod_idle(gpio1_oh);
	}

	/*
	 * Update Suspend_State value to be used in sleep33xx.S to keep
	 * GPIO0 module enabled during standby for EVM-SK
	 */
	if (suspend_state == PM_SUSPEND_STANDBY)
		suspend_cfg_param_list[SUSPEND_STATE] = PM_STANDBY;
	else
		suspend_cfg_param_list[SUSPEND_STATE] = PM_DS0;

#if 0  /*by szf*/
	/*
	 * Keep Touchscreen module enabled during standby
	 * to enable wakeup from standby.
	 */

	if (suspend_state == PM_SUSPEND_STANDBY)
		writel(0x2, AM33XX_CM_WKUP_ADC_TSC_CLKCTRL);
#endif

	if (gfx_l3_clkdm && gfx_l4ls_clkdm) {
		clkdm_sleep(gfx_l3_clkdm);
		clkdm_sleep(gfx_l4ls_clkdm);
	}

	/* Try to put GFX to sleep */
	if (gfx_pwrdm)
		pwrdm_set_next_pwrst(gfx_pwrdm, PWRDM_POWER_OFF);
	else
		pr_err("Could not program GFX to low power state\n");

	omap3_intc_suspend();

	am33xx_standby_setup(suspend_state);

	writel(0x0, AM33XX_CM_MPU_MPU_CLKCTRL);

	ret = cpu_suspend(0, am33xx_do_sram_idle);

	writel(0x2, AM33XX_CM_MPU_MPU_CLKCTRL);

	if (gfx_pwrdm) {
		state = pwrdm_read_pwrst(gfx_pwrdm);
		if (state != PWRDM_POWER_OFF)
			pr_err("GFX domain did not transition to low power state\n");
		else
			pr_info("GFX domain entered low power state\n");
	}

	/* XXX: Why do we need to wakeup the clockdomains? */
	if(gfx_l3_clkdm && gfx_l4ls_clkdm) {
		clkdm_wakeup(gfx_l3_clkdm);
		clkdm_wakeup(gfx_l4ls_clkdm);
	}

	/*
	 * Touchscreen module was enabled during standby
	 * Disable it here.
	 */
	if (suspend_state == PM_SUSPEND_STANDBY)
		writel(0x0, AM33XX_CM_WKUP_ADC_TSC_CLKCTRL);

	/*
	 * Put USB module to idle on resume from standby
	 */

	if (suspend_state == PM_SUSPEND_STANDBY)
		omap_hwmod_idle(usb_oh);

	/*
	 * Put RTC module to idle on resume from standby
	 * for PG2.x.
	 */
	if ((omap_rev() >= AM335X_REV_ES2_0) && (suspend_state == PM_SUSPEND_STANDBY))
		omap_hwmod_idle(rtc_oh);

	ret = am33xx_verify_lp_state(ret);

	/*
	 * Enable the GPIO module. Once the driver is
	 * fully adapted to runtime PM this will go away
	 */
	/*suspend_enter
	 * During standby, GPIO was not disabled. Hence no
	 * need to enable it here.
	 */
	if ((suspend_cfg_param_list[EVM_ID] != EVM_SK) &&
			(suspend_state != PM_SUSPEND_STANDBY))
		omap_hwmod_enable(gpio1_oh);

	return ret;
}

```

这个函数就是系统进入休眠之前所做的相关工作(板级相关的),不同的板子有不同的实现.可以看见里面把USB和GPIO还有触摸屏作为了唤醒源,具体实现,以后再分析.

### 第三个坑

是关于I2C设备的休眠问题,我们板子原来设计是可以单独开关I2C设备(PCA9555和BMI160),为了实现更低功耗,在休眠前就把这两个设备的电关掉,其实这样做是不对的,I2C设备的休眠唤醒应该在驱动中加,而且I2C设备是不建议单独开关电的.如果在休眠之前就把I2C设备电关掉,那在进入休眠的时候I2C总线会扫描设备，这样就会出错．