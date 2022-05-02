- 👋 Hi, I’m @Jujutsubot
- 👀 I’m interested in ...
- 🌱 I’m currently learning ...
- 💞️ I’m looking to collaborate on ...
- 📫 How to reach me ...

<!---
Jujutsubot/Jujutsubot is a ✨ special ✨ repository because its `README.md` (this file) appears on your GitHub profile.
You can click the Preview link to take a look at your changes.
--->

client.logger.level = 'error';
    console.log(banner);
    var commandHandler = new Map();
    try {
        var session = conn.restoreSession(config.STRING_SESSION)
        client.loadAuthInfo(session)
    } catch (err) {
        if (err instanceof TypeError || err.message === "given authInfo is null" || err instanceof SyntaxError) {
            console.log(
                chalk.redBright.bold("Incorrect Session String. Please authenticate again using command -> "),
                chalk.yellowBright.bold("npm start")
            );
            console.debug("[DEBUG] " + err);
            fs.writeFileSync('./config.env', `STRING_SESSION=""`);
            process.exit(0);
        }
        else {
            console.log(
                chalk.redBright.bold("SOMETHING WENT WRONG.\n"),
                chalk.redBright.bold("[DEBUG] " + err)
            );
            process.exit(0)
        }
    }

    client.on('connecting', async () => {
        console.log(chalk.yellowBright("[INFO] Connecting to WhatsApp..."));
    })

    client.on('open', async () => {
        console.log(chalk.yellowBright.bold("[INFO] Installing Plugins... Please wait."));
        var moduleFiles = fs.readdirSync(join(__dirname, 'modules')).filter((file) => file.endsWith('.js'))
        for (var file of moduleFiles) {
            try {
                const command = require(join(__dirname, 'modules', `${file}`));
                console.log(
                    chalk.magentaBright("[INFO] Successfully imported module"),
                    chalk.cyanBright.bold(`${file}`)
                )
                commandHandler.set(command.name, command);
            } catch (error) {
                console.log(
                    chalk.blueBright.bold("[INFO] Could not import module"),
                    chalk.redBright.bold(`${file}`)
                )
                console.log(`[ERROR] `, error);
                continue;
            }
        }
        console.log(chalk.green.bold("[INFO] Plugins Installed Successfully. The bot is ready to use."));
        console.log(chalk.yellowBright.bold("[INFO] Connecting to Database."));
        try {
            await sequelize.authenticate();
            console.log(chalk.greenBright.bold('[INFO] Connection has been established successfully.'));
        } catch (error) {
            console.error('[ERROR] Unable to connect to the database:', error);
        }
        console.log(chalk.yellowBright.bold("[INFO] Syncing Database..."));
        await sequelize.sync();
        console.log(chalk.greenBright.bold("[INFO] All models were synchronized successfully."));
        console.log(chalk.greenBright.bold("[INFO] Connected! Welcome to BotsApp"));
        client.sendMessage(
            client.user.jid,
            GENERAL.SUCCESSFUL_CONNECTION.format({
                worktype: config.WORK_TYPE,
            }),
            MessageType.text
        );
    })


    await client.connect();


    client.on('group-participants-update', async update => {
        // console.log("-------------------" + "GROUP PARTICIPANT UPDATE" + "-------------------");
        // console.log(update.participants);
        // console.log(update.action);
        // console.log(update.jid);
        var groupId = update.jid;

        try {
            if (update.action === 'add') {
                var enable = await Greetings.checkSettings(groupId, "welcome");
                if (enable === false || enable === "OFF") {
                    return;
                }
                var Msg = await Greetings.getMessage(groupId, "welcome");

                client.sendMessage(groupId, Msg.message, MessageType.text);
                return;
            }
            else if (update.action === 'remove') {
                var enable = await Greetings.checkSettings(groupId, "goodbye");
                if (enable === false || enable === "OFF") {
                    return;
                }
                var Msg = await Greetings.getMessage(groupId, "goodbye");

                client.sendMessage(groupId, Msg.message, MessageType.text);
                return;
            }
        }
        catch (err) {
            // console.log("Greeting message are off");
        }
    });

    client.on('chat-update', async chat => {
        if (!chat.hasNewMessage) return
        if (!chat.messages) return
        // console.log("-------------------------------------------")
        chat = chat.messages.all()[0];
        var sender = chat.key.remoteJid;
        const groupMetadata = sender.endsWith("@g.us") ? await client.groupMetadata(sender) : '';
        var BotsApp = wa.resolve(chat, client, groupMetadata);
        // console.log(BotsApp);
        if (BotsApp.isCmd) {
            let isBlacklist = await Blacklist.getBlacklistUser(BotsApp.sender, BotsApp.chatId);
            const cleared = await clearance(BotsApp, client, isBlacklist);
            if (!cleared) {
                return;
            }
            console.log(chalk.redBright.bold(`[INFO] ${BotsApp.commandName} command executed.`));
            const command = commandHandler.get(BotsApp.commandName);
            var args = BotsApp.body.trim().split(/\s+/).slice(1);
            // console.log("ARGS -> " + args);
            // args.forEach(arg => console.log("arg -> " + arg  + "  type -> " + typeof(arg)));
            // console.log("-------------------------------------------")
            if (!command) {
                client.sendMessage(BotsApp.chatId, "```Woops, invalid command! Use```  *.help*  ```to display the command list.```", MessageType.text);
                return;
            } else if (command && BotsApp.commandName == "help") {
                try {
                    command.handle(client, chat, BotsApp, args, commandHandler);
                    return;
                } catch (err) {
                    console.log(chalk.red("[ERROR] ", err));
                    return;
                }
            }
            try {
                command.handle(client, chat, BotsApp, args).catch(err => console.log("[ERROR] " + err));
            } catch (err) {
                console.log(chalk.red("[ERROR] ", err));
            }
        }
    })
}

main().catch(err => console.log('[ERROR] :
