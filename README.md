## Complete Recent Discord Quest (Updated February 2026)

> [!NOTE]
> This does not work in the browser for quests that require playing or streaming a game! Use the official [Discord desktop app](https://discord.com/download) for those quests.

### How to use this script:
1. Accept a quest in the Quests tab.
2. Open DevTools: Press <kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>I</kbd>.
3. Go to the `Console` tab.
4. Paste the code below and press Enter.

<details>
<summary>Click to expand the script</summary>

```js
(function() {
    delete window.$;
    let wpRequire = webpackChunkdiscord_app.push([[Symbol()], {}, r => r]);
    webpackChunkdiscord_app.pop();

    let ApplicationStreamingStore = Object.values(wpRequire.c).find(x => x?.exports?.Z?.__proto__?.getStreamerActiveStreamMetadata)?.exports?.Z;
    let RunningGameStore, QuestsStore, ChannelStore, GuildChannelStore, FluxDispatcher, api;
    if(!ApplicationStreamingStore) {
        ApplicationStreamingStore = Object.values(wpRequire.c).find(x => x?.exports?.A?.__proto__?.getStreamerActiveStreamMetadata).exports.A;
        RunningGameStore = Object.values(wpRequire.c).find(x => x?.exports?.Ay?.getRunningGames).exports.Ay;
        QuestsStore = Object.values(wpRequire.c).find(x => x?.exports?.A?.__proto__?.getQuest).exports.A;
        ChannelStore = Object.values(wpRequire.c).find(x => x?.exports?.A?.__proto__?.getAllThreadsForParent).exports.A;
        GuildChannelStore = Object.values(wpRequire.c).find(x => x?.exports?.Ay?.getSFWDefaultChannel).exports.Ay;
        FluxDispatcher = Object.values(wpRequire.c).find(x => x?.exports?.h?.__proto__?.flushWaitQueue).exports.h;
        api = Object.values(wpRequire.c).find(x => x?.exports?.Bo?.get).exports.Bo;
    } else {
        RunningGameStore = Object.values(wpRequire.c).find(x => x?.exports?.ZP?.getRunningGames).exports.ZP;
        QuestsStore = Object.values(wpRequire.c).find(x => x?.exports?.Z?.__proto__?.getQuest).exports.Z;
        ChannelStore = Object.values(wpRequire.c).find(x => x?.exports?.Z?.__proto__?.getAllThreadsForParent).exports.Z;
        GuildChannelStore = Object.values(wpRequire.c).find(x => x?.exports?.ZP?.getSFWDefaultChannel).exports.ZP;
        FluxDispatcher = Object.values(wpRequire.c).find(x => x?.exports?.Z?.__proto__?.flushWaitQueue).exports.Z;
        api = Object.values(wpRequire.c).find(x => x?.exports?.tn?.get).exports.tn;	
    }

    const supportedTasks = ["WATCH_VIDEO", "PLAY_ON_DESKTOP", "STREAM_ON_DESKTOP", "PLAY_ACTIVITY", "WATCH_VIDEO_ON_MOBILE"];
    let quests = [...QuestsStore.quests.values()].filter(x => x.userStatus?.enrolledAt && !x.userStatus?.completedAt && new Date(x.config.expiresAt).getTime() > Date.now() && supportedTasks.find(y => Object.keys((x.config.taskConfig ?? x.config.taskConfigV2).tasks).includes(y)));
    let isApp = typeof DiscordNative !== "undefined";

    if(quests.length === 0) {
        console.log("%c😔 No active uncompleted quests found!", "color: #ed4245; font-weight: bold;");
    } else {
        console.log(`%c🎉 Found ${quests.length} active quest(s)! Starting automation...`, "color: #7289DA; font-weight: bold;");

        let doJob = function() {
            const quest = quests.pop();
            if(!quest) return;

            const pid = Math.floor(Math.random() * 30000) + 1000;
            const applicationId = quest.config.application.id;
            const applicationName = quest.config.application.name;
            const questName = quest.config.messages.questName;
            const taskConfig = quest.config.taskConfig ?? quest.config.taskConfigV2;
            const taskName = supportedTasks.find(x => taskConfig.tasks[x] != null);
            const secondsNeeded = taskConfig.tasks[taskName].target;
            let secondsDone = quest.userStatus?.progress?.[taskName]?.value ?? 0;
            const remaining = secondsNeeded - secondsDone;

            console.log(`%c🚀 Starting: ${questName} (${taskName}) – ${remaining}s needed`, "color: #7289DA; font-weight: bold;");

            let lastUpdateLog = Date.now() - 15000; // allow immediate initial log
            let updateProgress = (current, force = false) => {
                const now = Date.now();
                if (force || now - lastUpdateLog >= 10000) {
                    const percentage = Math.min(100, Math.round((current / secondsNeeded) * 100));
                    const barLength = 30;
                    const filled = Math.round((percentage / 100) * barLength);
                    const bar = "▰".repeat(filled) + "▱".repeat(barLength - filled);
                    console.log(`%c  [${bar}] ${percentage}% (${current}/${secondsNeeded}s)`, "color: #7289DA; font-weight: bold;");
                    lastUpdateLog = now;
                }
            };

            updateProgress(secondsDone, true); // initial bar

            if(taskName === "WATCH_VIDEO" || taskName === "WATCH_VIDEO_ON_MOBILE") {
                const maxFuture = 10, speed = 7, interval = 1;
                const enrolledAt = new Date(quest.userStatus.enrolledAt).getTime();
                let completed = false;
                let fn = async () => {
                    while(true) {
                        const maxAllowed = Math.floor((Date.now() - enrolledAt)/1000) + maxFuture;
                        const diff = maxAllowed - secondsDone;
                        const timestamp = secondsDone + speed;
                        if(diff >= speed) {
                            const res = await api.post({url: `/quests/${quest.id}/video-progress`, body: {timestamp: Math.min(secondsNeeded, timestamp + Math.random())}});
                            completed = res.body.completed_at != null;
                            secondsDone = Math.min(secondsNeeded, timestamp);
                            updateProgress(secondsDone);
                        }
                        if(timestamp >= secondsNeeded) {
                            updateProgress(secondsNeeded, true);
                            break;
                        }
                        await new Promise(resolve => setTimeout(resolve, interval * 1000));
                    }
                    if(!completed) {
                        await api.post({url: `/quests/${quest.id}/video-progress`, body: {timestamp: secondsNeeded}});
                        updateProgress(secondsNeeded, true);
                    }
                    console.log(`%c✅ Quest completed: ${questName}!`, "color: #43b581; font-weight: bold;");
                    doJob();
                };
                fn();
                console.log(`Spoofing video for ${questName}...`);
            } else if(taskName === "PLAY_ON_DESKTOP") {
                if(!isApp) {
                    console.log(`%c⚠️  Non-video quests no longer work in browser. Skipping ${questName}!`, "color: #faa61a;");
                    doJob(); // skip to next quest
                } else {
                    api.get({url: `/applications/public?application_ids=${applicationId}`}).then(res => {
                        const appData = res.body[0];
                        const exeName = appData.executables.find(x => x.os === "win32").name.replace(">","");
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
                        };
                        const realGames = RunningGameStore.getRunningGames();
                        const realGetRunningGames = RunningGameStore.getRunningGames;
                        const realGetGameForPID = RunningGameStore.getGameForPID;
                        RunningGameStore.getRunningGames = () => [fakeGame];
                        RunningGameStore.getGameForPID = () => fakeGame;
                        FluxDispatcher.dispatch({type: "RUNNING_GAMES_CHANGE", removed: realGames, added: [fakeGame], games: [fakeGame]});
                        updateProgress(secondsDone, true);
                        console.log(`Spoofed game: ${applicationName}. Waiting ~${Math.ceil(remaining / 60)} minutes.`);
                        let fn = data => {
                            let progress = quest.config.configVersion === 1 ? data.userStatus.streamProgressSeconds : Math.floor(data.userStatus.progress.PLAY_ON_DESKTOP.value);
                            updateProgress(progress);
                            if(progress >= secondsNeeded) {
                                updateProgress(secondsNeeded, true);
                                console.log(`%c✅ Quest completed: ${questName}!`, "color: #43b581; font-weight: bold;");
                                RunningGameStore.getRunningGames = realGetRunningGames;
                                RunningGameStore.getGameForPID = realGetGameForPID;
                                FluxDispatcher.dispatch({type: "RUNNING_GAMES_CHANGE", removed: [fakeGame], added: [], games: []});
                                FluxDispatcher.unsubscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", fn);
                                doJob();
                            }
                        };
                        FluxDispatcher.subscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", fn);
                    });
                }
            } else if(taskName === "STREAM_ON_DESKTOP") {
                if(!isApp) {
                    console.log(`%c⚠️  Non-video quests no longer work in browser. Skipping ${questName}!`, "color: #faa61a;");
                    doJob(); // skip to next
                } else {
                    let realFunc = ApplicationStreamingStore.getStreamerActiveStreamMetadata;
                    ApplicationStreamingStore.getStreamerActiveStreamMetadata = () => ({id: applicationId, pid, sourceName: null});
                    updateProgress(secondsDone, true);
                    console.log(`Spoofed stream: ${applicationName}. Waiting ~${Math.ceil(remaining / 60)} minutes.`);
                    let fn = data => {
                        let progress = quest.config.configVersion === 1 ? data.userStatus.streamProgressSeconds : Math.floor(data.userStatus.progress.STREAM_ON_DESKTOP.value);
                        updateProgress(progress);
                        if(progress >= secondsNeeded) {
                            updateProgress(secondsNeeded, true);
                            console.log(`%c✅ Quest completed: ${questName}!`, "color: #43b581; font-weight: bold;");
                            ApplicationStreamingStore.getStreamerActiveStreamMetadata = realFunc;
                            FluxDispatcher.unsubscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", fn);
                            doJob();
                        }
                    };
                    FluxDispatcher.subscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", fn);
                }
            } else if(taskName === "PLAY_ACTIVITY") {
                const channelId = ChannelStore.getSortedPrivateChannels()[0]?.id ?? Object.values(GuildChannelStore.getAllGuilds()).find(x => x != null && x.VOCAL?.length > 0)?.VOCAL[0].channel.id;
                const streamKey = `call:${channelId}:1`;
                let fn = async () => {
                    console.log(`Completing PLAY_ACTIVITY for ${questName}...`);
                    updateProgress(secondsDone, true);
                    while(true) {
                        const res = await api.post({url: `/quests/${quest.id}/heartbeat`, body: {stream_key: streamKey, terminal: false}});
                        const progress = res.body.progress.PLAY_ACTIVITY.value;
                        updateProgress(progress);
                        await new Promise(resolve => setTimeout(resolve, 20000));
                        if(progress >= secondsNeeded) {
                            await api.post({url: `/quests/${quest.id}/heartbeat`, body: {stream_key: streamKey, terminal: true}});
                            updateProgress(secondsNeeded, true);
                            break;
                        }
                    }
                    console.log(`%c✅ Quest completed: ${questName}!`, "color: #43b581; font-weight: bold;");
                    doJob();
                };
                fn();
            }
        };
        doJob();
    }
})();
