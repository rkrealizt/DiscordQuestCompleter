## DiscordQuestCompleter

> [!NOTE]
> This does not works in browser for non-video quests! For stream/play quests use the desktop app!

> [!NOTE]
> When doing stream quests, you need at least 1 other account in the vc!

How to use this script:
1. Accept a quest under User Settings -> Gift Inventory
2. Press <kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>I</kbd> to open DevTools
3. Go to the `Console` tab
4. Paste the following code and hit enter:

<details>
	<summary>Click to expand code block</summary>
	
```js
let wpRequire;
window.webpackChunkdiscord_app.push([
  [Math.random()],
  {},
  (req) => {
    wpRequire = req;
  },
]);

let ApplicationStreamingStore = Object.values(wpRequire.c).find(
  (x) => x?.exports?.Z?.getStreamerActiveStreamMetadata,
).exports.Z;
let RunningGameStore = Object.values(wpRequire.c).find(
  (x) => x?.exports?.ZP?.getRunningGames,
).exports.ZP;
let QuestsStore = Object.values(wpRequire.c).find(
  (x) => x?.exports?.Z?.getQuest,
).exports.Z;
let ChannelStore = Object.values(wpRequire.c).find(
  (x) => x?.exports?.Z?.getAllThreadsForParent,
).exports.Z;
let GuildChannelStore = Object.values(wpRequire.c).find(
  (x) => x?.exports?.ZP?.getSFWDefaultChannel,
).exports.ZP;
let FluxDispatcher = Object.values(wpRequire.c).find(
  (x) => x?.exports?.Z?.flushWaitQueue,
).exports.Z;
let api = Object.values(wpRequire.c).find((x) => x?.exports?.tn?.get).exports
  .tn;

let quest = [...QuestsStore.quests.values()].find(
  (x) =>
    x.id !== "1248385850622869556" &&
    x.userStatus?.enrolledAt &&
    !x.userStatus?.completedAt &&
    new Date(x.config.expiresAt).getTime() > Date.now(),
);
let isApp = navigator.userAgent.includes("Electron/");
if (!quest) {
  console.log("You don't have any uncompleted quests!");
} else {
  const pid = Math.floor(Math.random() * 30000) + 1000;

  const applicationId = quest.config.application.id;
  const applicationName = quest.config.application.name;
  const taskName = [
    "WATCH_VIDEO",
    "PLAY_ON_DESKTOP",
    "STREAM_ON_DESKTOP",
    "PLAY_ACTIVITY",
  ].find((x) => quest.config.taskConfig.tasks[x] != null);
  const secondsNeeded = quest.config.taskConfig.tasks[taskName].target;
  const secondsDone = quest.userStatus?.progress?.[taskName]?.value ?? 0;

  if (taskName === "WATCH_VIDEO") {
    const tolerance = 2,
      speed = 10;
    const diff = Math.floor(
      (Date.now() - new Date(quest.userStatus.enrolledAt).getTime()) / 1000,
    );
    const startingPoint = Math.min(
      Math.max(Math.ceil(secondsDone), diff),
      secondsNeeded,
    );
    let fn = async () => {
      for (let i = startingPoint; i <= secondsNeeded; i += speed) {
        try {
          await api.post({
            url: `/quests/${quest.id}/video-progress`,
            body: { timestamp: Math.min(secondsNeeded, i + Math.random()) },
          });
        } catch (ex) {
          console.log("Failed to send increment of", i, ex.message);
        }
        await new Promise((resolve) => setTimeout(resolve, tolerance * 1000));
      }
      if ((secondsNeeded - secondsDone) % speed !== 0) {
        await api.post({
          url: `/quests/${quest.id}/video-progress`,
          body: { timestamp: secondsNeeded },
        });
      }
      console.log("Quest completed!");
    };
    fn();
    console.log(
      `Spoofing video for ${applicationName}. Wait for ${Math.ceil(((secondsNeeded - startingPoint) / speed) * tolerance)} more seconds.`,
    );
  } else if (taskName === "PLAY_ON_DESKTOP") {
    if (!isApp) {
      console.log(
        "This no longer works in browser for non-video quests. Use the desktop app to complete the",
        applicationName,
        "quest!",
      );
    }

    api
      .get({ url: `/applications/public?application_ids=${applicationId}` })
      .then((res) => {
        const appData = res.body[0];
        const exeName = appData.executables
          .find((x) => x.os === "win32")
          .name.replace(">", "");

        const games = RunningGameStore.getRunningGames();
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
        games.push(fakeGame);
        FluxDispatcher.dispatch({
          type: "RUNNING_GAMES_CHANGE",
          removed: [],
          added: [fakeGame],
          games: games,
        });

        let fn = (data) => {
          let progress =
            quest.config.configVersion === 1
              ? data.userStatus.streamProgressSeconds
              : Math.floor(data.userStatus.progress.PLAY_ON_DESKTOP.value);
          console.log(`Quest progress: ${progress}/${secondsNeeded}`);

          if (progress >= secondsNeeded) {
            console.log("Quest completed!");

            const idx = games.indexOf(fakeGame);
            if (idx > -1) {
              games.splice(idx, 1);
              FluxDispatcher.dispatch({
                type: "RUNNING_GAMES_CHANGE",
                removed: [fakeGame],
                added: [],
                games: [],
              });
            }
            FluxDispatcher.unsubscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", fn);
          }
        };
        FluxDispatcher.subscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", fn);

        console.log(
          `Spoofed your game to ${applicationName}. Wait for ${Math.ceil((secondsNeeded - secondsDone) / 60)} more minutes.`,
        );
      });
  } else if (taskName === "STREAM_ON_DESKTOP") {
    if (!isApp) {
      console.log(
        "This no longer works in browser for non-video quests. Use the desktop app to complete the",
        applicationName,
        "quest!",
      );
    }

    let realFunc = ApplicationStreamingStore.getStreamerActiveStreamMetadata;
    ApplicationStreamingStore.getStreamerActiveStreamMetadata = () => ({
      id: applicationId,
      pid,
      sourceName: null,
    });

    let fn = (data) => {
      let progress =
        quest.config.configVersion === 1
          ? data.userStatus.streamProgressSeconds
          : Math.floor(data.userStatus.progress.STREAM_ON_DESKTOP.value);
      console.log(`Quest progress: ${progress}/${secondsNeeded}`);

      if (progress >= secondsNeeded) {
        console.log("Quest completed!");

        ApplicationStreamingStore.getStreamerActiveStreamMetadata = realFunc;
        FluxDispatcher.unsubscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", fn);
      }
    };
    FluxDispatcher.subscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", fn);

    console.log(
      `Spoofed your stream to ${applicationName}. Stream any window in vc for ${Math.ceil((secondsNeeded - secondsDone) / 60)} more minutes.`,
    );
    console.log(
      "Remember that you need at least 1 other person to be in the vc!",
    );
  } else if (taskName === "PLAY_ACTIVITY") {
    const channelId =
      ChannelStore.getSortedPrivateChannels()[0]?.id ??
      Object.values(GuildChannelStore.getAllGuilds()).find(
        (x) => x != null && x.VOCAL.length > 0,
      ).VOCAL[0].channel.id;
    const streamKey = `call:${channelId}:1`;

    let fn = async () => {
      console.log(
        "Completing quest",
        applicationName,
        "-",
        quest.config.messages.questName,
      );

      while (true) {
        const res = await api.post({
          url: `/quests/${quest.id}/heartbeat`,
          body: { stream_key: streamKey, terminal: false },
        });
        const progress = res.body.progress.PLAY_ACTIVITY.value;
        console.log(`Quest progress: ${progress}/${secondsNeeded}`);

        await new Promise((resolve) => setTimeout(resolve, 20 * 1000));

        if (progress >= secondsNeeded) {
          await api.post({
            url: `/quests/${quest.id}/heartbeat`,
            body: { stream_key: streamKey, terminal: true },
          });
          break;
        }
      }

      console.log("Quest completed!");
    };
    fn();
  }
}
```
</details>

5. Follow the instructions displayed in the console based on your quest type:  
   - If the quest requires you to **play** a game, simply wait and let the process complete automatically.  
   - If the quest requires you to **stream** a game, join a voice channel with a friend or an alternate account and stream any window.  

7. Wait for approximately **15 minutes**.  

8. Once completed, go to **User Settings → Gift Inventory** to claim your reward!  

You can monitor your progress in two ways:  
- Check the `Quest progress:` messages printed in the **Console tab**.  
- Refresh the **Gift Inventory** tab in settings to see updates.

## FAQ

**Q: Ctrl + Shift + I doesn't work**  
**A:** You can try one of the following options:  
- Download the [PTB client](https://discord.com/api/downloads/distributions/app/installers/latest?channel=ptb&platform=win&arch=x64).  
- Use [this guide](https://www.reddit.com/r/discordapp/comments/sc61n3/comment/hu4fw5x/) to enable DevTools on the stable version of Discord.  

---

**Q: I get an error saying "Unauthorized"**  
**A:** Discord has updated their platform, preventing this script from working in browsers. To fix this:  
- Use the **Discord desktop app** instead.  
- Alternatively, find an extension that allows you to modify your User-Agent and include the string `Electron/` in it.  

Additionally, Discord now checks the number of participants in a voice channel. Make sure to have at least **one other account** in the voice channel with you.
