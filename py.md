# My First Markdown
## Python
```python
class Task:
    def __init__(self, shared, name):
        self.taskName = name
        self.shared = shared
        self.configLoaded = False
        self.pid = 0
        self.retry = 0
        self.logger = logging.getLogger('Task:%s' % (name))
        if DEBUG:
            print(self.shared)

    def loadConfig(self, configFile):
        try:
            fd = open(configFile)
        except:
            self.logger.fatal(u'打开配置文件"%s"失败!' % (configFile))
            return False
        try:
            r = json.load(fd)
        except:
            self.logger.fatal(u'配置文件"%s"格式有误!' % (configFile))
            return False
        if self.taskName not in r:
            self.logger.fatal(u'配置文件"%s"中不存在本任务的配置!' % (configFile))
            return False
        essential = ['pidFile', 'interval', 'alertEmail']
        for attr in essential:
            try:
                setattr(self, attr, r[self.taskName][attr])
                del r[self.taskName][attr]
            except:
                self.logger.fatal(u'配置中缺少必选配置"%s"!' % (attr))
                return False
        self.logger.info(u'读取必选配置完成!')
        self.configLoaded = True
        optional = r[self.taskName].keys()
        for attr in optional:
            setattr(self, attr, r[self.taskName][attr])
            self.logger.info(u'读取可选配置"%s"!' % (attr))
        self.logger.info(u'读取任务配置完成!')
        fd.close()
        return True

    def __sendAlertMail(self, text):
        if self.configLoaded:
            msg = MIMEText(u'%s %s\nsend by pyMonit' % (datetime.datetime.now(), text), 'plain', 'utf-8')
            msg['Subject'] = Header(u'%s:%s 告警信息' % (self.shared['config']['hostName'],self.taskName), 'utf-8')
            msg['From'] = self.shared['config']['smtpUserName']
            msg['To'] = self.alertEmail
            smtp = smtplib.SMTP()
            smtp.connect(self.shared['config']['smtpHost'])
            smtp.login(self.shared['config']['smtpUserName'], self.shared['config']['smtpPassword'])
            smtp.sendmail(self.shared['config']['smtpUserName'], self.alertEmail, msg.as_string())
            self.logger.info(u'告警邮件发送成功!')
            smtp.quit()

    def test(self):
        if self.configLoaded:
            self.logger.info('xxxx')
            self.start()

    def __getPid(self):
        if self.configLoaded:
            try:
                pidFd = open(self.pidFile)
            except:
                self.logger.error(u'找不到pid文件:%s!' % (self.pidFile))
                return False
            try:
                tPid = int(pidFd.readline())
            except:
                self.logger.error(u'pid文件格式错误!')
                return False
            if self.pid == 0 or self.pid == tPid:
                self.pid = tPid
                self.logger.info(u'当前pid为:%d' % (self.pid))
                return True
            else:
                self.logger.warning(u'pid发生变化!(%d => %d)' % (self.pid, tPid))
                self.pid = tPid
                return True

    def __checkStatus(self):
        if self.configLoaded:
            if self.__getPid():
                if not self.__parseProcStatusFile():
                    self.__sendAlertMail(u'任务"%s"已停止运行!' % (self.taskName))
                    if not hasattr(self,'retryLimit') or self.retry < self.retryLimit:
                        self.args = shlex.split(self.startCommand.encode())
                        self.shared['pOpenLock'].acquire()
                        self.p = subprocess.Popen(self.args)
                        self.shared['pOpenLock'].release()
                        self.retry += 1
                        self.logger.info(u'正在尝试重新启动任务!')
                        self.pid = self.p.pid
                        self.start()
                    else:
                        self.logger.error(u'超出重新启动次数上限,放弃!')
                        self.__sendAlertMail(u'超出重新启动次数上限,放弃任务"%s"!' %(self.taskName))
                else:
                    self.logger.info(u'任务运行正常!')
                    self.retry = 0
                    self.start()
            else:
                self.__sendAlertMail(u'任务"%s"无法被监控!' % (self.taskName))


    def __parseProcStatusFile(self):
        if self.configLoaded:
            self.statusPath = '/proc/%d/status' % (self.pid)
            if not os.access(self.statusPath, os.F_OK):
                self.logger.error(u'任务不存在!')
                return False
            else:
                return True

    def start(self):
        if self.configLoaded:
            self.timer = threading.Timer(self.interval, self.__checkStatus)
            self.timer.start()
            self.timer.setName(self.taskName)
    
    def stop(self, signalNum, frame):
        self.timer.cancel()
```