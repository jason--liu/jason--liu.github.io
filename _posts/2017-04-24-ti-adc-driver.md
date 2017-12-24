---
layout: post
title: "Ti AM335x ADC Driver"
modified:
categories: blog
excerpt:
tags: [driver adc]
image:
  feature:
---

```shell
内核版本:linux-3.2.0
单板:AM3352
```

由于这个版本的内核相对比较老,没有采用设备树,而是采用板级文件的方式来构建单板,所以分析起来相对简单.TI的ADC驱动采用的还是平台总线的驱动方式,即driver-platformbus-device.

先来看看platform-device.

```c
static struct adc_data am335x_adc_data = {
	.adc_channels = 1,//ADC通道数
};

static struct mfd_tscadc_board tscadc = {
	.adc_init = &am335x_adc_data,
};

static void mfd_tscadc_init(int evm_id, int profile)
{
	int err;

	err = am33xx_register_mfd_tscadc(&tscadc);
	if (err)
		pr_err("failed to register touchscreen device\n");
}
```

关键是am33xx_register_mfd_tscadc函数,来看看里面做了什么,由于只是分析框架,所以去掉了一些不大相关的代码.

``` c

int __init am33xx_register_mfd_tscadc(struct mfd_tscadc_board *pdata)
{
	...
    ...
	char *oh_name = "adc_tsc";
	char *dev_name = "ti_tscadc";
	...
	pdev = omap_device_build(dev_name, id, oh, pdata,
			sizeof(struct mfd_tscadc_board), NULL, 0, 0);
  	...
	return 0;
}

```

再看看omap_device_build函数做了什么.

```c
struct platform_device *omap_device_build(const char *pdev_name, int pdev_id,
				      struct omap_hwmod *oh, void *pdata,
				      int pdata_len,
				      struct omap_device_pm_latency *pm_lats,
				      int pm_lats_cnt, int is_early_device)
{
	struct omap_hwmod *ohs[] = { oh };

	if (!oh)
		return ERR_PTR(-EINVAL);

	return omap_device_build_ss(pdev_name, pdev_id, ohs, 1, pdata,
				    pdata_len, pm_lats, pm_lats_cnt,
				    is_early_device);
}
```

任然看不出什么特别的名堂,继续跟代码看看omap_device_build_ss函数又做了什么

```c
struct platform_device *omap_device_build_ss(const char *pdev_name, int pdev_id,
					 struct omap_hwmod **ohs, int oh_cnt,
					 void *pdata, int pdata_len,
					 struct omap_device_pm_latency *pm_lats,
					 int pm_lats_cnt, int is_early_device)
{
	int ret = -ENOMEM;
	struct platform_device *pdev;
	struct omap_device *od;

	if (!ohs || oh_cnt == 0 || !pdev_name)
		return ERR_PTR(-EINVAL);

	if (!pdata && pdata_len > 0)
		return ERR_PTR(-EINVAL);

	pdev = platform_device_alloc(pdev_name, pdev_id);
	if (!pdev) {
		ret = -ENOMEM;
		goto odbs_exit;
	}

	/* Set the dev_name early to allow dev_xxx in omap_device_alloc */
	if (pdev->id != -1)
		dev_set_name(&pdev->dev, "%s.%d", pdev->name,  pdev->id);
	else
		dev_set_name(&pdev->dev, "%s", pdev->name);

	od = omap_device_alloc(pdev, ohs, oh_cnt, pm_lats, pm_lats_cnt);
	if (!od)
		goto odbs_exit1;

	ret = platform_device_add_data(pdev, pdata, pdata_len);
	if (ret)
		goto odbs_exit2;

	if (is_early_device)
		ret = omap_early_device_register(pdev);
	else
		ret = omap_device_register(pdev);
	if (ret)
		goto odbs_exit2;

	return pdev;

odbs_exit2:
	omap_device_delete(od);
odbs_exit1:
	platform_device_put(pdev);
odbs_exit:

	pr_err("omap_device: %s: build failed (%d)\n", pdev_name, ret);

	return ERR_PTR(ret);
}

```

到这局势就逐渐明朗了,哈哈.先分配一个platform_device结构体,然后填充这个结构体,最后再注册它.由于is_early_device传的值是0,所以又会调用omap_device_register函数.

```c
int omap_device_register(struct platform_device *pdev)
{
	pr_debug("omap_device: %s: registering\n", pdev->name);

	pdev->dev.parent = &omap_device_parent;
	pdev->dev.pm_domain = &omap_device_pm_domain;
	return platform_device_add(pdev);
}
```

到这,platform_device基本就清晰了,而它的name就是ti_tscadc,根据platform_device--platform_bus--platform_driver三者之间的关系,有一个"ti_tscadc"的platform_device,也应该有同名的platform_device.所以,我们搜索ti_tscadc即可.果然在driver/mfd/ti_tscadc.c中发现了这个结构体

```c
static struct platform_driver ti_tscadc_driver = {
	.driver = {
		.name   = "ti_tscadc",
		.owner	= THIS_MODULE,
	},
	.probe	= ti_tscadc_probe,
	.remove	= __devexit_p(ti_tscadc_remove),
	.suspend = tscadc_suspend,
	.resume = tscadc_resume,
};
```

我们只关心它的probe函数.

```c
static int __devinit ti_tscadc_probe(struct platform_device *pdev)
{
	struct ti_tscadc_dev	*tscadc;
	struct resource		*res;
	struct clk		*clk;
	struct mfd_tscadc_board	*pdata = pdev->dev.platform_data;
	struct mfd_cell		*cell;
	int			err, ctrl, children = 0;
	int			clk_value, clock_rate;
	int			tsc_wires = 0, adc_channels = 0, total_channels;

	if (!pdata) {
		dev_err(&pdev->dev, "Could not find platform data\n");
		return -EINVAL;
	}
	//获取平台数据,这里我们只设置了adc_channels
	if (pdata->adc_init)
		adc_channels = pdata->adc_init->adc_channels;

	if (pdata->tsc_init)
		tsc_wires = pdata->tsc_init->wires;

	total_channels = tsc_wires + adc_channels;

	if (total_channels > 8) {
		dev_err(&pdev->dev, "Number of i/p channels more than 8\n");
		return -EINVAL;
	}

	res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
	if (!res) {
		dev_err(&pdev->dev, "no memory resource defined.\n");
		return -EINVAL;
	}

	/* Allocate memory for device */
	tscadc = kzalloc(sizeof(struct ti_tscadc_dev), GFP_KERNEL);
	if (!tscadc) {
		dev_err(&pdev->dev, "failed to allocate memory.\n");
		return -ENOMEM;
	}

	res = request_mem_region(res->start, resource_size(res),
			pdev->name);
	if (!res) {
		dev_err(&pdev->dev, "failed to reserve registers.\n");
		err = -EBUSY;
		goto err_free_mem;
	}

	tscadc->tscadc_base = ioremap(res->start, resource_size(res));
	if (!tscadc->tscadc_base) {
		dev_err(&pdev->dev, "failed to map registers.\n");
		err = -ENOMEM;
		goto err_release_mem;
	}

	tscadc->irq = platform_get_irq(pdev, 0);
	if (tscadc->irq < 0) {
		dev_err(&pdev->dev, "no irq ID is specified.\n");
		return -ENODEV;
	}

	tscadc->dev = &pdev->dev;

	pm_runtime_enable(&pdev->dev);
	pm_runtime_get_sync(&pdev->dev);

	/*
	 * The TSC_ADC_Subsystem has 2 clock domains
	 * OCP_CLK and ADC_CLK.
	 * The ADC clock is expected to run at target of 3MHz,
	 * and expected to capture 12-bit data at a rate of 200 KSPS.
	 * The TSC_ADC_SS controller design assumes the OCP clock is
	 * at least 6x faster than the ADC clock.
	 */
	clk = clk_get(&pdev->dev, "adc_tsc_fck");
	if (IS_ERR(clk)) {
		dev_err(&pdev->dev, "failed to get TSC fck\n");
		err = PTR_ERR(clk);
		goto err_fail;
	}
	clock_rate = clk_get_rate(clk);
	clk_put(clk);
	clk_value = clock_rate / ADC_CLK;
	/* TSCADC_CLKDIV needs to be configured to the value minus 1 */
	clk_value = clk_value - 1;
	tscadc_writel(tscadc, TSCADC_REG_CLKDIV, clk_value);

	/* Set the control register bits */
	ctrl = TSCADC_CNTRLREG_STEPCONFIGWRT |
			TSCADC_CNTRLREG_STEPID;
	if (pdata->tsc_init)
		ctrl |= TSCADC_CNTRLREG_4WIRE |
				TSCADC_CNTRLREG_TSCENB;
	tscadc_writel(tscadc, TSCADC_REG_CTRL, ctrl);

	/* Set register bits for Idle Config Mode */
	if (pdata->tsc_init)
		tscadc_idle_config(tscadc);

	/* Enable the TSC module enable bit */
	ctrl = tscadc_readl(tscadc, TSCADC_REG_CTRL);
	ctrl |= TSCADC_CNTRLREG_TSCSSENB;
	tscadc_writel(tscadc, TSCADC_REG_CTRL, ctrl);

	/* TSC Cell */
	if (pdata->tsc_init) {
		cell = &tscadc->cells[children];
		cell->name = "tsc";
		cell->platform_data = tscadc;
		cell->pdata_size = sizeof(*tscadc);
		children++;
	}

	/* ADC Cell */
	if (pdata->adc_init) {
		cell = &tscadc->cells[children];
		cell->name = "tiadc";
		cell->platform_data = tscadc;
		cell->pdata_size = sizeof(*tscadc);
		children++;
	}

	err = mfd_add_devices(&pdev->dev, pdev->id, tscadc->cells,
			children, NULL, 0);
	if (err < 0)
		goto err_fail;

	device_init_wakeup(&pdev->dev, true);
	platform_set_drvdata(pdev, tscadc);
	return 0;

err_fail:
	pm_runtime_put_sync(&pdev->dev);
	pm_runtime_disable(&pdev->dev);
	iounmap(tscadc->tscadc_base);
err_release_mem:
	release_mem_region(res->start, resource_size(res));
	mfd_remove_devices(tscadc->dev);
err_free_mem:
	platform_set_drvdata(pdev, NULL);
	kfree(tscadc);
	return err;
}
```

这个函数做的相对比较复杂,具体看注释.注意到mfd_add_devices这个函数,

```c

int mfd_add_devices(struct device *parent, int id,
		    struct mfd_cell *cells, int n_devs,
		    struct resource *mem_base,
		    int irq_base)
{
	int i;
	int ret = 0;
	atomic_t *cnts;

	/* initialize reference counting for all cells */
	cnts = kcalloc(sizeof(*cnts), n_devs, GFP_KERNEL);
	if (!cnts)
		return -ENOMEM;

	for (i = 0; i < n_devs; i++) {
		atomic_set(&cnts[i], 0);
		cells[i].usage_count = &cnts[i];
		ret = mfd_add_device(parent, id, cells + i, mem_base, irq_base);
		if (ret)
			break;
	}

	if (ret)
		mfd_remove_devices(parent);

	return ret;
}
//再看看mfd_add_device函数

static int mfd_add_device(struct device *parent, int id,
			  const struct mfd_cell *cell,
			  struct resource *mem_base,
			  int irq_base)
{
	struct resource *res;
	struct platform_device *pdev;
	int ret = -ENOMEM;
	int r;

	pdev = platform_device_alloc(cell->name, id + cell->id);
	if (!pdev)
		goto fail_alloc;

	res = kzalloc(sizeof(*res) * cell->num_resources, GFP_KERNEL);
	if (!res)
		goto fail_device;

	pdev->dev.parent = parent;

	if (cell->pdata_size) {
		ret = platform_device_add_data(pdev,
					cell->platform_data, cell->pdata_size);
		if (ret)
			goto fail_res;
	}

	ret = mfd_platform_add_cell(pdev, cell);
	if (ret)
		goto fail_res;

	for (r = 0; r < cell->num_resources; r++) {
		res[r].name = cell->resources[r].name;
		res[r].flags = cell->resources[r].flags;

		/* Find out base to use */
		if ((cell->resources[r].flags & IORESOURCE_MEM) && mem_base) {
			res[r].parent = mem_base;
			res[r].start = mem_base->start +
				cell->resources[r].start;
			res[r].end = mem_base->start +
				cell->resources[r].end;
		} else if (cell->resources[r].flags & IORESOURCE_IRQ) {
			res[r].start = irq_base +
				cell->resources[r].start;
			res[r].end   = irq_base +
				cell->resources[r].end;
		} else {
			res[r].parent = cell->resources[r].parent;
			res[r].start = cell->resources[r].start;
			res[r].end   = cell->resources[r].end;
		}

		if (!cell->ignore_resource_conflicts) {
			ret = acpi_check_resource_conflict(res);
			if (ret)
				goto fail_res;
		}
	}

	ret = platform_device_add_resources(pdev, res, cell->num_resources);
	if (ret)
		goto fail_res;

	ret = platform_device_add(pdev);
	if (ret)
		goto fail_res;

	if (cell->pm_runtime_no_callbacks)
		pm_runtime_no_callbacks(&pdev->dev);

	kfree(res);

	return 0;

fail_res:
	kfree(res);
fail_device:
	platform_device_put(pdev);
fail_alloc:
	return ret;
}
```

又回到了platform_device--platform_bus--platform_device那一套,而name就是"tiadc".搜索它,发现它在

driver/staging/iio/adc/ti_adc.c

```c
static struct platform_driver tiadc_driver = {
	.driver = {
		.name   = "tiadc",
		.owner = THIS_MODULE,
	},
	.probe          = tiadc_probe,
	.remove         = __devexit_p(tiadc_remove),
	.suspend = adc_suspend,
	.resume = adc_resume,
};
```

在看看它的probe函数.

```c
static int __devinit tiadc_probe(struct platform_device *pdev)
{
	struct iio_dev		*idev;
	struct adc_device	*adc_dev = NULL;
	struct ti_tscadc_dev	*tscadc_dev = pdev->dev.platform_data;
	struct mfd_tscadc_board	*pdata;
	int			err;

	pdata = (struct mfd_tscadc_board *)tscadc_dev->dev->platform_data;
	if (!pdata || !pdata->adc_init)  {
		dev_err(tscadc_dev->dev, "Could not find platform data\n");
		return -EINVAL;
	}

	idev = iio_allocate_device(sizeof(struct adc_device));
	if (idev == NULL) {
		dev_err(&pdev->dev, "failed to allocate iio device.\n");
		err = -ENOMEM;
		goto err_ret;
	}
	adc_dev = iio_priv(idev);

	tscadc_dev->adc = adc_dev;
	adc_dev->mfd_tscadc = tscadc_dev;
	adc_dev->idev = idev;
	adc_dev->channels = pdata->adc_init->adc_channels;
	adc_dev->irq = tscadc_dev->irq;

	idev->dev.parent = &pdev->dev;
	idev->name = dev_name(&pdev->dev);
	idev->modes = INDIO_DIRECT_MODE;
	idev->info = &tiadc_info;

	/* by default driver comes up with oneshot mode */
	adc_step_config(adc_dev, adc_dev->is_continuous_mode);

	/* program FIFO threshold to value minus 1 */
	adc_writel(adc_dev, TSCADC_REG_FIFO1THR, FIFO1_THRESHOLD);

	err = tiadc_channel_init(idev, adc_dev);
	if (err < 0)
		goto err_free_device;

	init_waitqueue_head(&adc_dev->wq_data_avail);

	err = request_irq(adc_dev->irq, tiadc_irq, IRQF_SHARED,
		idev->name, idev);
	if (err)
		goto err_cleanup_channels;

	err = tiadc_config_sw_ring(idev);
	if (err)
		goto err_free_irq;

	err = iio_buffer_register(idev,
			idev->channels, idev->num_channels);
	if (err < 0)
		goto err_free_sw_rb;

	err = iio_device_register(idev);
	if (err)
		goto err_unregister;

	dev_info(&pdev->dev, "attached adc driver\n");
	platform_set_drvdata(pdev, idev);

	return 0;

err_unregister:
	iio_buffer_unregister(idev);
err_free_sw_rb:
	iio_sw_rb_free(idev->buffer);
err_free_irq:
	free_irq(adc_dev->irq, idev);
err_cleanup_channels:
	tiadc_channel_remove(idev);
err_free_device:
	iio_free_device(idev);
err_ret:
	return err;
}
```

其实从这个函数开始,就涉及到了Linux的IIO子系统,有空再来分析,未完待续.