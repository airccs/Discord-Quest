# Discord-Quest
Создан для быстрого выполнения заданий Discord без запуска игр или потоков.

> [!NOTE]
> Это больше не работает в браузере!
> 
> Это больше не работает, если вы один в vc! Кто-то еще должен присоединиться к вам!
>

> [!WARNING]
> Теперь есть два типа квестов ("стрим" и "игра")! Обращайте внимание на инструкции!
>

Как использовать этот скрипт:
1. Примите квест в разделе Настройки пользователя -> Склад подарков
2. Нажмите <kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>I</kbd> чтобы открыть инструменты разработчика
3. Перейдите в вкладку `Console` 
4. Вставьте следующий код и нажмите клавишу Enter:
<details>
	<summary>Click to expand</summary>
	
```js
let wpRequire;
window.webpackChunkdiscord_app.push([[ Math.random() ], {}, (req) => { wpRequire = req; }]);

let ApplicationStreamingStore, RunningGameStore, QuestsStore, ExperimentStore, FluxDispatcher, api
if(window.GLOBAL_ENV.SENTRY_TAGS.buildId === "366c746173a6ca0a801e9f4a4d7b6745e6de45d4") {
	ApplicationStreamingStore = Object.values(wpRequire.c).find(x => x?.exports?.default?.getStreamerActiveStreamMetadata).exports.default;
	RunningGameStore = Object.values(wpRequire.c).find(x => x?.exports?.default?.getRunningGames).exports.default;
	QuestsStore = Object.values(wpRequire.c).find(x => x?.exports?.default?.getQuest).exports.default;
	ExperimentStore = Object.values(wpRequire.c).find(x => x?.exports?.default?.getGuildExperiments).exports.default;
	FluxDispatcher = Object.values(wpRequire.c).find(x => x?.exports?.default?.flushWaitQueue).exports.default;
	api = Object.values(wpRequire.c).find(x => x?.exports?.getAPIBaseURL).exports.HTTP;
} else {
	ApplicationStreamingStore = Object.values(wpRequire.c).find(x => x?.exports?.Z?.getStreamerActiveStreamMetadata).exports.Z;
	RunningGameStore = Object.values(wpRequire.c).find(x => x?.exports?.ZP?.getRunningGames).exports.ZP;
	QuestsStore = Object.values(wpRequire.c).find(x => x?.exports?.Z?.getQuest).exports.Z;
	ExperimentStore = Object.values(wpRequire.c).find(x => x?.exports?.Z?.getGuildExperiments).exports.Z;
	FluxDispatcher = Object.values(wpRequire.c).find(x => x?.exports?.Z?.flushWaitQueue).exports.Z;
	api = Object.values(wpRequire.c).find(x => x?.exports?.tn?.get).exports.tn;
}

let quest = [...QuestsStore.quests.values()].find(x => x.id !== "1245082221874774016" && x.userStatus?.enrolledAt && !x.userStatus?.completedAt && new Date(x.config.expiresAt).getTime() > Date.now())
let isApp = navigator.userAgent.includes("Electron/")
if(!isApp) {
	console.log("This no longer works in browser. Use the desktop app!")
} else if(!quest) {
	console.log("You don't have any uncompleted quests!")
} else {
	const pid = Math.floor(Math.random() * 30000) + 1000
	
	let applicationId, applicationName, secondsNeeded, secondsDone, canPlay
	if(quest.config.configVersion === 1) {
		applicationId = quest.config.applicationId
		applicationName = quest.config.applicationName
		secondsNeeded = quest.config.streamDurationRequirementMinutes * 60
		secondsDone = quest.userStatus?.streamProgressSeconds ?? 0
		canPlay = quest.config.variants.includes(2)
	} else if(quest.config.configVersion === 2) {
		applicationId = quest.config.application.id
		applicationName = quest.config.application.name
		canPlay = ExperimentStore.getUserExperimentBucket("2024-04_quest_playtime_task") > 0 && quest.config.taskConfig.tasks["PLAY_ON_DESKTOP"]
		const taskName = canPlay ? "PLAY_ON_DESKTOP" : "STREAM_ON_DESKTOP"
		secondsNeeded = quest.config.taskConfig.tasks[taskName].target
		secondsDone = quest.userStatus?.progress?.[taskName]?.value ?? 0
	}

	if(canPlay) {
		api.get({url: `/applications/public?application_ids=${applicationId}`}).then(res => {
			const appData = res.body[0]
			const exeName = appData.executables.find(x => x.os === "win32").name.replace(">","")
			
			const games = RunningGameStore.getRunningGames()
			const fakeGame = {
				cmdLine: `C:\\Program Files\\${appData.name}\\${exeName}`,
				exeName,
				exePath: `c:/program files/${appData.name.toLowerCase()}/${exeName}`,
				hidden: false,
				isLauncher: false,
				id: applicationId,
				name: appData.name,
				pid: pid,
				pidPath: [pid],
				processName: appData.name,
				start: Date.now(),
			}
			games.push(fakeGame)
			FluxDispatcher.dispatch({type: "RUNNING_GAMES_CHANGE", removed: [], added: [fakeGame], games: games})
			
			let fn = data => {
				let progress = quest.config.configVersion === 1 ? data.userStatus.streamProgressSeconds : Math.floor(data.userStatus.progress.PLAY_ON_DESKTOP.value)
				console.log(`Quest progress: ${progress}/${secondsNeeded}`)
				
				if(progress >= secondsNeeded) {
					console.log("Quest completed!")
					
					const idx = games.indexOf(fakeGame)
					if(idx > -1) {
						games.splice(idx, 1)
						FluxDispatcher.dispatch({type: "RUNNING_GAMES_CHANGE", removed: [fakeGame], added: [], games: []})
					}
					FluxDispatcher.unsubscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", fn)
				}
			}
			FluxDispatcher.subscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", fn)
			
			console.log(`Spoofed your game to ${applicationName}. Wait for ${Math.ceil((secondsNeeded - secondsDone) / 60)} more minutes.`)
		})
	} else {
		let realFunc = ApplicationStreamingStore.getStreamerActiveStreamMetadata
		ApplicationStreamingStore.getStreamerActiveStreamMetadata = () => ({
			id: applicationId,
			pid,
			sourceName: null
		})
		
		let fn = data => {
			let progress = quest.config.configVersion === 1 ? data.userStatus.streamProgressSeconds : Math.floor(data.userStatus.progress.STREAM_ON_DESKTOP.value)
			console.log(`Quest progress: ${progress}/${secondsNeeded}`)
			
			if(progress >= secondsNeeded) {
				console.log("Quest completed!")
				
				ApplicationStreamingStore.getStreamerActiveStreamMetadata = realFunc
				FluxDispatcher.unsubscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", fn)
			}
		}
		FluxDispatcher.subscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", fn)
		
		console.log(`Spoofed your stream to ${applicationName}. Stream any window in vc for ${Math.ceil((secondsNeeded - secondsDone) / 60)} more minutes.`)
		console.log("Remember that you need at least 1 other person to be in the vc!")
	}
}
```
</details>

5. Следуйте инструкциям в зависимости от типа квеста.
- Если в квесте сказано "играть", вы можете просто ждать и ничего не делать.
- Если в задании сказано "стримить" игру, присоединитесь к vc с другом или вашим твинком и стримите в любом окне.
7. Подождите 15 минут
8. Теперь вы можете получить награду в Настройки пользователя -> Инвентарь подарков!

Вы можете отслеживать прогресс, глядя на `Quest progress:` отпечатки на вкладке "Console" или снова открыв вкладку "Инвентарь подарков" в настройках.

## FAQ

**Вопрос: Ctrl + Shift + не работает**

Ответ: Загрузите [ptb client](https://discord.com/api/downloads/distributions/app/installers/latest?channel=ptb&platform=win&arch=x64), или использовать [это](https://www.reddit.com/r/discordapp/comments/sc61n3/comment/hu4fw5x/) чтобы включить инструменты разработчика в стабильной версии


**Вопрос: Я получаю ошибку "Unauthorized"**

Ответ: Discord исправил скрипт, который не работает в браузерах. Используйте настольное приложение или найдите какое-нибудь расширение, которое позволит вам изменить User-Agent и добавьте строку `Electron/` в любом месте.

Разработчики также начали проверять, сколько человек находится в голосовом канале, поэтому убедитесь, что вы присоединились к нему как минимум на 1 другом аккаунте.

**Вопрос: Я получаю другую ошибку**

Ответ: Убедитесь, что вы правильно скопировали/вставили скрипт и выполнили все шаги.
