<!--
author: vlean
date: 2016-10-19
title: PHP多进程
tags: php,多进程,进程通信,php扩展
category: php
status: publish
summary: php多进程的开启和调用。
-->

php进程控制的扩展有两个`pcntl`和`posix`，分别用于进程控制和信号控制，`pcntl`扩展只能在cli模式下使用，`posix`windows平台不支持。

## 多进程示例

```php
if (substr(php_sapi_name(), 0, 3) !== 'cli') {
  die("This Programe can only be run in CLI mode");
}

$maxProcessNum = 10;
$subProcessNum = 0;

while(true){
  if($subProcessNum > $maxProcessNum){
    sleep(1);
    continue;
  }

  $pid = pcntl_fork();
  if($pid == 0){
    assignTask();
  }elseif($pid==-1){
    show("create subProcess fail");
  }else{
    $subProcessNum++;
    show("subProcess num:".$subProcessNum);
    if($subProcessNum>= $maxProcessNum){
      show("wait subProcess");
      pcntl_wait($status);
      $subProcessNum--;
    }
  }
}

function show($msg,$level="DEBUG"){
  echo sprintf("[%s] %s | %s\n",$level,posix_getpid(),$msg);
}

function assignTask(){
  $needTime = rand(1,10);
  show("Task Begin");
  sleep($needTime);
  show("Task End");
}
```

## 执行结果
```bash
[DEBUG] 10770 | subProcess num:1
[DEBUG] 10771 | Task Begin
[DEBUG] 10770 | subProcess num:2
[DEBUG] 10772 | Task Begin
[DEBUG] 10770 | subProcess num:3
[DEBUG] 10773 | Task Begin
[DEBUG] 10770 | subProcess num:4
[DEBUG] 10774 | Task Begin
[DEBUG] 10770 | subProcess num:5
...
```
