---
title: 获取用户 IP
date: 2022-07-28 11:34:25
tags:
---


```java
function getIp()
{
	$ips = array();
	if ($_SERVER['HTTP_CLIENT_IP']) {
		$ips[] = $_SERVER['HTTP_CLIENT_IP'];
	}
	if ($_SERVER['HTTP_X_FORWARDED_FOR']) {
		$tmp = explode(', ', $_SERVER['HTTP_X_FORWARDED_FOR']);
		$ips = array_merge($ips, $tmp);
	}
	if ($_SERVER['REMOTE_ADDR']) {
		$ips[] = $_SERVER['REMOTE_ADDR'];
	}
	$ip = '';
	foreach ($ips as $k => $v) {
		if(!preg_match('/^(10|172\.16|192\.168)\./', $v) && strtolower($v) != 'unknown') {
			$ip = $v;
			break;
		}
	}
	return $ip;
}
```
