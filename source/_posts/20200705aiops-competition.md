---
title: 20200305aiops_competition复赛
date: 2020-07-05 22:34:31
tags:
- AIOps
- 论文
- AnomalyDetection
categories:
- AIOps
---



### aiops比赛消费kafka数据

1，在自己电脑上用终端远程登录服务器： ssh user@ip

2，输入docker image 查看镜像情况：

3，启动docker

```shell
docker run -it aiops_0705 /bin/bash
```

4，Python执行consumer.py ，consumer.py的kafka的数据怎么取的：对于平台指标和业务指标，解析后的data的主要内容为从指标类型到数据列表的字典；对于调用链数据，则只有一条调用链数据。对data的访问就是对python中dict的访问。

如下：打印的数据就是kafka的实时数据了，可以实时获取到：

```python
# 前面class部分，PlatformIndex(item)，BusinessIndex(item)， Trace(data)见Reference的github

def main():
	'''Consume data and react'''
	# Check authorities
	assert AVAILABLE_TOPICS <= CONSUMER.topics(), 'Please contact admin'

	submit([['docker_003', 'container_cpu_used']])
	i = 0
	for message in CONSUMER:
		i += 1
		data = json.loads(message.value.decode('utf8'))
		if message.topic == 'platform-index':
			# data['body'].keys() is supposed to be
			# ['os_linux', 'db_oracle_11g', 'mw_redis', 'mw_activemq',
			#  'dcos_container', 'dcos_docker']
			data = {
				'timestamp': data['timestamp'],
				'body': {
					stack: [PlatformIndex(item) for item in data['body'][stack]]
					for stack in data['body']
				},
			}
			timestamp = data['timestamp']
			for stack in data['body']:
				print(i, message.topic, timestamp, stack)
				for item in data['body'][stack]:
					# like class
					print (item.name)

		elif message.topic == 'business-index':
			# data['body'].keys() is supposed to be ['esb', ]
			data = {
				'startTime': data['startTime'],
				'body': {
					key: [BusinessIndex(item) for item in data['body'][key]]
					for key in data['body']
				},
			}
			timestamp = data['startTime']
			for key in data['body']:
				print (i, message.topic, timestamp, key)
				# data['body'][key] is a class
				for item in data['body'][key]:
					print (item.service_name)
		else:  # message.topic == 'trace'
			data = {
				'startTime': data['startTime'],
				'body': Trace(data),
			}
			timestamp = data['startTime']
			# one data
			print(i, message.topic, timestamp, data['body'].pid)



if __name__ == '__main__':
	main()
```



### Reference

1，https://github.com/NetManAIOps/aiops2020-judge/tree/master/final

2，[kafka原理](https://www.cnblogs.com/sujing/p/10960832.html)

3，[dockerfile的使用](https://www.runoob.com/docker/docker-dockerfile.html)