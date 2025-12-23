## Complete Accepted Quests

> [!NOTE]
> This script is intended for **educational and testing purposes only**. Following these instructions should not get you banned, but there is always a small risk. Always use at your own discretion and never share sensitive account information.

> This does **not** work in a browser for quests that require playing a game. Use the [desktop app](https://discord.com/download) along with [Vencord client](https://vencord.dev/) instead.

---

### How to Use This Script

1. Accept any quest under **Discover → Quests**.
2. Press <kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>I</kbd> to open **DevTools**.
3. Go to the **Console** tab.
4. Paste the following code and press Enter:

<details>
<summary>Click to expand</summary>

```js
// --- YOUR SCRIPT START ---
let wpRequire = webpackChunkdiscord_app.push([[Symbol()], {}, r => r]);
webpackChunkdiscord_app.pop();

let QuestsStore = Object.values(wpRequire.c)
  .find(m => m?.exports?.Z?.__proto__?.getQuest)?.exports.Z;

let RunningGameStore = Object.values(wpRequire.c)
  .find(m => m?.exports?.ZP?.getRunningGames)?.exports.ZP;

let ApplicationStreamingStore = Object.values(wpRequire.c)
  .find(m => m?.exports?.Z?.__proto__?.getStreamerActiveStreamMetadata)?.exports.Z;

let FluxDispatcher = Object.values(wpRequire.c)
  .find(m => m?.exports?.Z?.__proto__?.flushWaitQueue)?.exports.Z;

let api = Object.values(wpRequire.c)
  .find(m => m?.exports?.tn?.get)?.exports.tn;

const supportedTasks = [
  "WATCH_VIDEO",
  "PLAY_ON_DESKTOP",
  "STREAM_ON_DESKTOP",
  "PLAY_ACTIVITY",
  "WATCH_VIDEO_ON_MOBILE"
];

let quests = [...QuestsStore.quests.values()].filter(q => {
  const taskConfig = q.config.taskConfig ?? q.config.taskConfigV2;
  const hasTask = supportedTasks.some(t => Object.keys(taskConfig.tasks).includes(t));
  const incomplete = !q.userStatus?.completedAt && new Date(q.config.expiresAt).getTime() > Date.now();
  return hasTask && incomplete;
});

if (!quests.length) {
  console.log("No uncompleted quests found!");
} else {
  console.log(`Found ${quests.length} uncompleted quests!`);
  const isApp = typeof DiscordNative !== "undefined";

  const doJob = async () => {
    const quest = quests.pop();
    if (!quest) return;

    const pid = Math.floor(Math.random() * 30000) + 1000;
    const appId = quest.config.application.id;
    const appName = quest.config.application.name;
    const questName = quest.config.messages.questName;
    const taskConfig = quest.config.taskConfig ?? quest.config.taskConfigV2;
    const taskName = supportedTasks.find(t => taskConfig.tasks[t] != null);
    const secondsNeeded = taskConfig.tasks[taskName].target;
    let secondsDone = quest.userStatus?.progress?.[taskName]?.value ?? 0;

    if (taskName.includes("VIDEO")) {
      const enrolledAt = new Date(quest.userStatus.enrolledAt).getTime();
      let completed = false;

      while (secondsDone < secondsNeeded) {
        const maxFuture = 10, speed = 7;
        const maxAllowed = Math.floor((Date.now() - enrolledAt)/1000) + maxFuture;
        const diff = maxAllowed - secondsDone;
        const timestamp = secondsDone + speed;

        if (diff >= speed) {
          const res = await api.post({
            url: `/quests/${quest.id}/video-progress`,
            body: { timestamp: Math.min(secondsNeeded, timestamp + Math.random()) }
          });
          completed = res.body.completed_at != null;
          secondsDone = Math.min(secondsNeeded, timestamp);
        }

        await new Promise(r => setTimeout(r, 1000));
      }

      if (!completed) {
        await api.post({ url: `/quests/${quest.id}/video-progress`, body: { timestamp: secondsNeeded } });
      }

      console.log(`Video quest "${questName}" completed!`);
      doJob();
    }

    else if (taskName === "PLAY_ON_DESKTOP") {
      if (!isApp) return console.log(`Desktop app required for "${questName}"`);

      const res = await api.get({ url: `/applications/public?application_ids=${appId}` });
      const exeName = res.body[0].executables.find(e => e.os === "win32").name;

      const fakeGame = {
        cmdLine: `C:\\Program Files\\${appName}\\${exeName}`,
        exeName,
        exePath: `c:/program files/${appName.toLowerCase()}/${exeName}`,
        hidden: false,
        isLauncher: false,
        id: appId,
        name: appName,
        pid,
        pidPath: [pid],
        processName: appName,
        start: Date.now(),
      };

      const realGames = RunningGameStore.getRunningGames();
      const realGetRunningGames = RunningGameStore.getRunningGames;
      const realGetGameForPID = RunningGameStore.getGameForPID;

      RunningGameStore.getRunningGames = () => [fakeGame];
      RunningGameStore.getGameForPID = (p) => [fakeGame].find(g => g.pid === p);

      FluxDispatcher.dispatch({
        type: "RUNNING_GAMES_CHANGE",
        removed: realGames,
        added: [fakeGame],
        games: [fakeGame]
      });

      console.log(`Spoofed game "${appName}" for quest "${questName}".`);

      const fn = data => {
        const progress = Math.floor(data.userStatus.progress.PLAY_ON_DESKTOP.value);
        console.log(`Progress: ${progress}/${secondsNeeded}`);
        if (progress >= secondsNeeded) {
          RunningGameStore.getRunningGames = realGetRunningGames;
          RunningGameStore.getGameForPID = realGetGameForPID;
          FluxDispatcher.dispatch({ type: "RUNNING_GAMES_CHANGE", removed: [fakeGame], added: [], games: [] });
          FluxDispatcher.unsubscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", fn);
          console.log(`Quest "${questName}" completed!`);
          doJob();
        }
      };

      FluxDispatcher.subscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", fn);
    }

    else if (taskName === "STREAM_ON_DESKTOP") {
      if (!isApp) return console.log(`Desktop app required for "${questName}"`);

      const realFunc = ApplicationStreamingStore.getStreamerActiveStreamMetadata;
      ApplicationStreamingStore.getStreamerActiveStreamMetadata = () => ({ id: appId, pid, sourceName: null });

      const fn = data => {
        const progress = Math.floor(data.userStatus.progress.STREAM_ON_DESKTOP.value);
        console.log(`Progress: ${progress}/${secondsNeeded}`);
        if (progress >= secondsNeeded) {
          ApplicationStreamingStore.getStreamerActiveStreamMetadata = realFunc;
          FluxDispatcher.unsubscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", fn);
          console.log(`Quest "${questName}" completed!`);
          doJob();
        }
      };

      FluxDispatcher.subscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", fn);
      console.log(`Spoofed stream for "${questName}".`);
    }
  };

  doJob();
}
```

</details>

5. Follow the printed instructions depending on the quest type:

   * **Video quests**: just wait, it will simulate progress.
   * **Stream quests**: join a VC with a friend or alt and stream any window.

6. Wait for the script to finish.

7. Claim your reward manually.

---

### FAQ

**Q: Can I get banned?**
A: There is always a small risk. Use this only for testing/educational purposes. Never share account credentials.

**Q: Ctrl+Shift+I doesn’t work**
A: Use the [PTB client](https://discord.com/api/downloads/distributions/app/installers/latest?channel=ptb&platform=win&arch=x64) or enable DevTools on stable using [this method](https://www.reddit.com/r/discordapp/comments/sc61n3/comment/hu4fw5x/).

**Q: Syntax errors?**
A: Disable browser translation, copy the script fully, and paste it in DevTools.

**Q: Can I auto accept quests or rewards?**
A: No. Most require a captcha, so it’s not fully automatable.

**Q: Can this be a Vencord plugin?**
A: No. The script often requires immediate updates, which is not feasible with plugin review cycles.

---

### Legal / Disclaimer

* This script is provided **as-is for educational purposes only**.
* I am **not responsible** for any account issues resulting from misuse.
* Use **only on accounts you own**.
* Do **not share sensitive information** such as tokens or passwords.
* By using this, you accept **all risk** of Discord action against your account.
* Following these instructions **should not get you banned**, but there is always a small risk.

---

### License

This project is licensed under the **MIT License**. See [LICENSE](https://github.com/mboonstra-X/discord/blob/main/LICENSE) for details.


[CREDITS](https://gist.github.com/aamiaa/204cd9d42013ded9faf646fae7f89fbb)
