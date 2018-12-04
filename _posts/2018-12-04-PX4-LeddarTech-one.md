---
layout: post
title:  "PX4 LeddarTech One"
categories: PX4
tags:  LeddarTech PX4  
author: Byte
---

* content
{:toc}


## 前言
LeddarTech 公司的产品开发资料在官网上找不到，都在技术支持网站中：[https://support.leddartech.com/login](https://support.leddartech.com/login)


## LeddarOne驱动

1 先请求：
```js
// 请求读数
static const uint8_t request_reading_msg[] = {
	MODBUS_SLAVE_ADDRESS,
	MODBUS_READING_FUNCTION,
	0, /* starting addr high byte */
	READING_START_ADDR,
	0, /* number of bytes to read high byte */
	READING_LEN,
	0x30, /* CRC low */
	0x09 /* CRC high */
};
bool leddar_one::_request()
{
	/* flush anything in RX buffer */
	tcflush(_fd, TCIFLUSH); // 刷新RX缓冲区中的任何东西。

	int r = ::write(_fd, request_reading_msg, sizeof(request_reading_msg)); // 写入从地址、读、开始地址、长度。
	return r == sizeof(request_reading_msg);
}

// 请求在这里处理
int leddar_one::_cycle()
{
	int ret = 0;
	const hrt_abstime now = hrt_absolute_time(); //绝对时间

	switch (_state) { // 初始化时候_state = state_waiting_reading;
	case state_waiting_reading: // 等待读数
		if (now > _timeout_usec) {
			if (_request()) { // 发送请求
				perf_begin(_sample_perf); //ROS 目前不是很明白
				_buffer_len = 0; // 清除buffer
				_state = state_reading_requested; // 发送请求后就开始读数据。
				_timeout_usec = now + COLLECT_USEC_TIMEOUT;
			}
		}

		break;

	case state_reading_requested: // 读数请求
		ret = _collect(); // 把数据publish出去

		if (ret == 1) { //为1代表已经读到数据
			perf_end(_sample_perf);  // ROS 目前不是很明白
			_state = state_waiting_reading; //改变状态为等待数据，去发送请求
			_timeout_usec = now + READING_USEC_PERIOD;

		} else {
			if (ret == 0 && now < _timeout_usec) {
				/* still waiting for reading */
				break;
			}

			if (ret == -1) {  // -1则为错误
				perf_count(_comm_error);

			} else {  // 0则为仍然等待读数
				perf_count(_collect_timeout_perf);
			}

			perf_cancel(_sample_perf);
			_state = state_waiting_reading;
			_timeout_usec = 0;
			return _cycle(); // 重新cycle
		}
	}

	return ret;
}
```
2 数据处理
 
```js

// 周期运转
void
leddar_one::cycle_trampoline(void *arg)
{
	leddar_one *dev = reinterpret_cast<leddar_one * >(arg);

	if (dev->_fd != -1) {
		dev->_cycle();

	} else {
		dev->_fd_open();
	}
    // 用Nuttx的work_queue把cycle_trampoline送入内核线程去执行。
	work_queue(HPWORK, &(dev->_work), (worker_t)&leddar_one::cycle_trampoline,
		   dev, USEC2TICK(WORK_USEC_INTERVAL));
}

void leddar_one::_publish(uint16_t distance_mm)
{
	struct distance_sensor_s report;

	report.timestamp = hrt_absolute_time(); // 获得时间轴。
	report.type = distance_sensor_s::MAV_DISTANCE_SENSOR_ULTRASOUND; // 超声波距离传感器。
	report.orientation = _rotation; // 方向在main中有写：ROTATION_DOWNWARD_FACING。/向下/
	report.current_distance = ((float)distance_mm / 1000.0f); // 在_collect()中可以得到距离信息。
	report.min_distance = MIN_DISTANCE; // 最小距离已经设定
	report.max_distance = MAX_DISTANCE; // 最大距离已经设定
	report.covariance = 0.0f; // 协方差在此设置
	report.signal_quality = -1; // 信号质量就这样给
	report.id = 0;

	_reports->force(&report); //此句是环形队列中,将项目强制放入缓冲区，如果没有空间，则丢弃旧项目.

	if (_topic == nullptr) {
		_topic = orb_advertise(ORB_ID(distance_sensor), &report); //若是话题为空指针，则广告话题

	} else {
		orb_publish(ORB_ID(distance_sensor), _topic, &report);//此次广告完后，下一次就直接发布出去
	}
}

// 发布在这里处理，这个函数在_cycle()里面处理
int leddar_one::_collect()
{
	struct reading_msg *msg;

	int r = ::read(_fd, _buffer + _buffer_len, sizeof(_buffer) - _buffer_len);

	if (r < 1) {  // 仍然等待着读数
		return 0;
	}

	_buffer_len += r;
	// buffer小于所要读的信息长度，则退出继续收。
	if (_buffer_len < sizeof(struct reading_msg)) {
		return 0;
	}

	msg = (struct reading_msg *)_buffer;
	// 若是地址不对函数不对，则返回 -1
	if (msg->slave_addr != MODBUS_SLAVE_ADDRESS || msg->function != MODBUS_READING_FUNCTION) {
		return -1;
	}

	const uint16_t crc16_calc = _calc_crc16(_buffer, _buffer_len - 2);
	// crc不对则返回 -1
	if (crc16_calc != msg->crc) {
		return -1;
	}

	/* NOTE: little-endian support only */
	// 仅仅支持小端在前
	_publish(msg->first_dist_high_byte << 8 | msg->first_dist_low_byte); //publish出去
	return 1;
}

```

3 其他

```js
// 读信息
struct __attribute__((__packed__)) reading_msg {
	uint8_t slave_addr;//从地址
	uint8_t function;
	uint8_t len;
	uint8_t low_timestamp_high_byte; //时间
	uint8_t low_timestamp_low_byte;
	uint8_t high_timestamp_high_byte;
	uint8_t high_timestamp_low_byte;
	uint8_t temp_high;// 温度
	uint8_t temp_low;
	uint8_t num_detections_high_byte;
	uint8_t num_detections_low_byte;
	uint8_t first_dist_high_byte;//第一个距离高字节
	uint8_t first_dist_low_byte;
	uint8_t first_amplitude_high_byte;// 振幅
	uint8_t first_amplitude_low_byte;
	uint8_t second_dist_high_byte;// 第二个距离
	uint8_t second_dist_low_byte;
	uint8_t second_amplitude_high_byte;// 振幅
	uint8_t second_amplitude_low_byte;
	uint8_t third_dist_high_byte;// 第三个距离
	uint8_t third_dist_low_byte;
	uint8_t third_amplitude_high_byte;// 振幅
	uint8_t third_amplitude_low_byte;
	uint16_t crc; /* little-endian */
};


// { "leddar_one", SCHED_PRIORITY_DEFAULT, 1200, leddar_one_main },
int leddar_one_main(int argc, char *argv[])
{
	int ch;
	int myoptind = 1;
	const char *myoptarg = nullptr; //
	const char *serial_port = "/dev/ttyS3"; // 串口号
	uint8_t rotation = distance_sensor_s::ROTATION_DOWNWARD_FACING; //得到此测距仪得的朝向。
	static leddar_one *inst = nullptr; // 函数赋给指针变量。
	// px4_getopt()是什么？  EOF为 end of file。
	while ((ch = px4_getopt(argc, argv, "d:r", &myoptind, &myoptarg)) != EOF) {
		switch (ch) {
		case 'd': // 端口号
			serial_port = myoptarg;
			break;

		case 'r': // 方向
			rotation = (uint8_t)atoi(myoptarg);
			break;

		default: // 报错
			help();
			return PX4_ERROR;
		}
	}

	if (myoptind >= argc) {
		help();
		return PX4_ERROR;
	}

	const char *verb = argv[myoptind];

	if (!strcmp(verb, "start")) { // 若是verb为start则实例化一个leddar_one
		inst = new leddar_one(DEVICE_PATH, serial_port, rotation);

		if (!inst) {
			PX4_ERR("No memory to allocate " NAME);
			return PX4_ERROR;
		}

		if (inst->init() != PX4_OK) {
			delete inst;
			return PX4_ERROR;
		}

		inst->start();//开始放入队列

	} else if (!strcmp(verb, "stop")) { // 停止则删除实例化
		delete inst;
		inst = nullptr;

	} else if (!strcmp(verb, "test")) { // 测试
		int fd = open(DEVICE_PATH, O_RDONLY);
		ssize_t sz;
		struct distance_sensor_s report;

		if (fd < 0) {
			PX4_ERR("Unable to open %s", DEVICE_PATH);
			return PX4_ERROR;
		}

		sz = read(fd, &report, sizeof(report));
		close(fd);

		if (sz != sizeof(report)) {
			PX4_ERR("No sample available in %s", DEVICE_PATH);
			return PX4_ERROR;
		}

		print_message(report);

	} else {
		help();
		return PX4_ERROR;
	}

	return PX4_OK;
}


int leddar_one::_fd_open()
{
	if (!_serial_port) {
		return -1;
	}

	_fd = ::open(_serial_port, O_RDWR | O_NOCTTY | O_NONBLOCK);

	if (_fd < 0) {
		return -1;
	}

	struct termios config;

	int r = tcgetattr(_fd, &config);

	if (r) {
		PX4_ERR("Unable to get termios from %s.", _serial_port);
		goto error;
	}
	// 配置串口
	/*
	 * clear: data bit size, two stop bits, parity, hardware flow control
	 * 数据位大小，两个停止位，奇偶校验，硬件流控制
	 */
	config.c_cflag &= ~(CSIZE | CSTOPB | PARENB | CRTSCTS);
	/* set: 8 data bits, enable receiver, ignore modem status lines */
	config.c_cflag |= (CS8 | CREAD | CLOCAL);
	/* turn off output processing */
	config.c_oflag = 0;
	/* clear: echo, echo new line, canonical input and extended input */
	config.c_lflag &= (ECHO | ECHONL | ICANON | IEXTEN);

	r = cfsetispeed(&config, B115200);
	r |= cfsetospeed(&config, B115200);

	if (r) {
		PX4_ERR("Unable to set baudrate");
		goto error;
	}

	r = tcsetattr(_fd, TCSANOW, &config);

	if (r) {
		PX4_ERR("Unable to set termios to %s", _serial_port);
		goto error;
	}

	return PX4_OK;

error:
	::close(_fd);
	_fd = -1;
	return PX4_ERROR;
}


ssize_t leddar_one::read(struct file *filp, char *buffer, size_t buflen)
{
	unsigned count = buflen / sizeof(struct distance_sensor_s);
	struct distance_sensor_s *rbuf = reinterpret_cast<struct distance_sensor_s *>(buffer);
	int ret = 0;

	if (count < 1) {
		return -ENOSPC; // No space left on device.
	}

	while (count--) {
		if (_reports->get(rbuf)) { // 头 = 尾，则返回false ，环形队列里。
			ret += sizeof(*rbuf);
			rbuf++;
		}
	}

	return ret ? ret : -EAGAIN; // Try again.
}

void
leddar_one::stop()
{
	work_cancel(HPWORK, &_work); // 取消此队列工作后，就不会再排队了，重新work_queue()才能再排队。
}

void leddar_one::start()
{
	/*
	 * file descriptor can only be accessed by the process that opened it
	 * so closing here and it will be opened from the High priority kernel
	 * process
	 */
	::close(_fd);
	_fd = -1;

	work_queue(HPWORK, &_work, (worker_t)&leddar_one::cycle_trampoline, this, USEC2TICK(WORK_USEC_INTERVAL)); // 开始排队
}


int leddar_one::_fd_open()
{
	if (!_serial_port) {
		return -1;
	}

	_fd = ::open(_serial_port, O_RDWR | O_NOCTTY | O_NONBLOCK);

	if (_fd < 0) {
		return -1;
	}

	struct termios config;

	int r = tcgetattr(_fd, &config);

	if (r) {
		PX4_ERR("Unable to get termios from %s.", _serial_port);
		goto error;
	}
	// 配置串口
	/*
	 * clear: data bit size, two stop bits, parity, hardware flow control
	 * 数据位大小，两个停止位，奇偶校验，硬件流控制
	 */
	config.c_cflag &= ~(CSIZE | CSTOPB | PARENB | CRTSCTS);
	/* set: 8 data bits, enable receiver, ignore modem status lines */
	config.c_cflag |= (CS8 | CREAD | CLOCAL);
	/* turn off output processing */
	config.c_oflag = 0;
	/* clear: echo, echo new line, canonical input and extended input */
	config.c_lflag &= (ECHO | ECHONL | ICANON | IEXTEN);

	r = cfsetispeed(&config, B115200);
	r |= cfsetospeed(&config, B115200);

	if (r) {
		PX4_ERR("Unable to set baudrate");
		goto error;
	}

	r = tcsetattr(_fd, TCSANOW, &config);

	if (r) {
		PX4_ERR("Unable to set termios to %s", _serial_port);
		goto error;
	}

	return PX4_OK;

error:
	::close(_fd);
	_fd = -1;
	return PX4_ERROR;
}

int leddar_one::init()
{
	if (CDev::init()) {
		PX4_ERR("Unable to initialize device\n");
		return -1;
	}

	_reports = new ringbuffer::RingBuffer(2, sizeof(distance_sensor_s));

	if (!_reports) {
		PX4_ERR("No memory to allocate RingBuffer");
		return -1;
	}

	if (_fd_open()) {
		return PX4_ERROR;
	}

	hrt_abstime timeout_usec, now;

	for (now = hrt_absolute_time(), timeout_usec = now + PROBE_USEC_TIMEOUT;
	     now < timeout_usec;
	     now = hrt_absolute_time()) {

		if (_cycle() > 0) {
			printf(NAME " found\n");
			return PX4_OK;
		}

		usleep(1000);
	}

	PX4_ERR("No readings from " NAME);

	return PX4_ERROR;
}
```