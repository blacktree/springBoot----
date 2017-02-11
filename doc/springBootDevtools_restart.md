 **SBdevtools的重启机制**
 
 一、本地监控文件变化自动重启
1. 读取 配置属性spring.devtools.restart.enabled 
2. 判断AgentReloader   isActive

``` stylus
	static {
		Set<String> agentClasses = new LinkedHashSet<String>();
		agentClasses.add("org.zeroturnaround.javarebel.Integration");
		agentClasses.add("org.zeroturnaround.javarebel.ReloaderFactory");
		AGENT_CLASSES = Collections.unmodifiableSet(agentClasses);
	}
```
 isActive
 

``` stylus
	public static boolean isActive() {
		return isActive(null) || isActive(AgentReloader.class.getClassLoader())
				|| isActive(ClassLoader.getSystemClassLoader());
	}

	private static boolean isActive(ClassLoader classLoader) {
		for (String agentClass : AGENT_CLASSES) {
			if (ClassUtils.isPresent(agentClass, classLoader)) {
				return true;
			}
		}
		return false;
	}
```
3. DevToolsSettings 设置哪些URL（restart.include.、restart.exclude.）里的文件发生变化，则重启 
4. 启动一个ClassPathFileSystemWatcher 监控URL，收集FileSnapshot（新增、删除、File.lastModified()）

``` stylus
	public void start() {
		synchronized (this.monitor) {
			saveInitialSnapshots();
			if (this.watchThread == null) {
				Map<File, FolderSnapshot> localFolders = new HashMap<File, FolderSnapshot>();
				localFolders.putAll(this.folders);
				this.watchThread = new Thread(new Watcher(this.remainingScans,
						new ArrayList<FileChangeListener>(this.listeners),
						this.triggerFilter, this.pollInterval, this.quietPeriod,
						localFolders));
				this.watchThread.setName("File Watcher");
				this.watchThread.setDaemon(this.daemon);
				this.watchThread.start();
			}
		}
	}
```
当存在变化，publishEvent(ClassPathChangedEvent)

``` stylus
	@Override
	public void onChange(Set<ChangedFiles> changeSet) {
		boolean restart = isRestartRequired(changeSet);
		publishEvent(new ClassPathChangedEvent(this, changeSet, restart));
	}

	private void publishEvent(ClassPathChangedEvent event) {
		this.eventPublisher.publishEvent(event);
		if (event.isRestartRequired() && this.fileSystemWatcherToStop != null) {
			this.fileSystemWatcherToStop.stop();
		}
	}
```


5. 最终RestartClassLoader  

还有种方式是通过HTTP请求  ，将需要修改的文件通过HTTP上传，再重新加载
org.springframework.boot.devtools.restart.server.HttpRestartServer

``` stylus
	public void handle(ServerHttpRequest request, ServerHttpResponse response)
			throws IOException {
		try {
			Assert.state(request.getHeaders().getContentLength() > 0, "No content");
			ObjectInputStream objectInputStream = new ObjectInputStream(
					request.getBody());
			ClassLoaderFiles files = (ClassLoaderFiles) objectInputStream.readObject();
			objectInputStream.close();
			this.server.updateAndRestart(files);
			response.setStatusCode(HttpStatus.OK);
		}
		catch (Exception ex) {
			logger.warn("Unable to handler restart server HTTP request", ex);
			response.setStatusCode(HttpStatus.INTERNAL_SERVER_ERROR);
		}
	}
```
