---
layout:     post
title:      "Laravel安装报错"
date:       2016-10-26 17:53:00
author:     "Harry"
header-img: "img/post-bg-2015.jpg"
tags:
    - PHP
---


今天同事安装Laravel报错，帮忙看了一下，错误如下：

![Laravel安装报错](http://oeii54s39.bkt.clouddn.com/laravel-install-error.png)

通过分析代码，最终找到了报错的位置vendor\symfony\process\Process.php:380,下面贴一下出错的函数
    public function wait(callable $callback = null)
    {
        $this->requireProcessIsStarted(__FUNCTION__);

        $this->updateStatus(false);

        if (null !== $callback) {
            if (!$this->processPipes->haveReadSupport()) {
                $this->stop(0);
                throw new \LogicException('Pass the callback to the Process:start method or enableOutput to use a callback with Process::wait');
            }
            $this->callback = $this->buildCallback($callback);
        }

        do {
            $this->checkTimeout();
            $running = '\\' === DIRECTORY_SEPARATOR ? $this->isRunning() : $this->processPipes->areOpen();
            $this->readPipes($running, '\\' !== DIRECTORY_SEPARATOR || !$running);
        } while ($running);

        while ($this->isRunning()) {
            usleep(1000);
        }

        if ($this->processInformation['signaled'] && $this->processInformation['termsig'] !== $this->latestSignal) {
            throw new RuntimeException(sprintf('The process has been signaled with signal "%s".', $this->processInformation['termsig']));
        }

        return $this->exitcode;
    }

从源代码可以看出，这是composer执行script后，等待返回结果时报错了，返回的termsig为11

google了一下signal 11，发现signal 11 指的是 segmentation fault [https://www.centos.org/docs/5/html/5.2/Installation_Guide/ch14s01s01.html](https://www.centos.org/docs/5/html/5.2/Installation_Guide/ch14s01s01.html "具体戳这里")

segmentation fault 这个错误很眼熟，因为同事的linux环境无论执行什么指令都会报这个错误，所以可以确定问题就出在这里

为了再次确认，手动执行了一下

php -r "file_exists('.env') || copy('.env.example', '.env');"

确实是会报Segmentation fault

所以由此看来确实是由于这个问题照成的，不过这个问题就暂不深究了，最后解决方法是手动执行了laraval中Composer.json里面的scripts指令
