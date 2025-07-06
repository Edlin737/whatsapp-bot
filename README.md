```js
const { default: makeWASocket, useMultiFileAuthState, DisconnectReason, fetchLatestBaileysVersion } = require('@whiskeysockets/baileys');
const P = require('pino');

const startBot = async () => {
    const { state, saveCreds } = await useMultiFileAuthState('auth');
    const { version } = await fetchLatestBaileysVersion();
    const sock = makeWASocket({
        version,
        logger: P({ level: 'silent' }),
        printQRInTerminal: true,
        auth: state
    });

    sock.ev.on('creds.update', saveCreds);

    // Welcome message
    sock.ev.on('group-participants.update', async (update) => {
        if (update.action === 'add') {
            const number = update.participants[0].split('@')[0];
            const welcomeText = `👋 Welcome to the group, @${number}!`;
            await sock.sendMessage(update.id, { text: welcomeText, mentions: update.participants });
        }
    });

    // Messages listener
    sock.ev.on('messages.upsert', async ({ messages }) => {const msg = messages[0];
        if (!msg.message || msg.key.fromMe) return;

        const from = msg.key.remoteJid;
        const text = msg.message.conversation || msg.message.extendedTextMessage?.text || '';

        // Tag All
        if (text === '!tagall') {
            const metadata = await sock.groupMetadata(from);
            const mentions = metadata.participants.map(p => p.id);
            await sock.sendMessage(from, { text: '🔔 Tagging all members...', mentions });
        }

        // Anti-link
        if (text.includes('chat.whatsapp.com') && !msg.key.fromMe) {
            const metadata = await sock.groupMetadata(from);
            const admins = metadata.participants.filter(p => p.admin).map(p => p.id);
            const sender = msg.key.participant;
            if (!admins.includes(sender)) {
                await sock.sendMessage(from, { text: '🚫 Group link not allowed. You will be removed.' });
                await sock.groupParticipantsUpdate(from, [sender], 'remove');
            }
        }
    });

    sock.ev.on('connection.update', ({ connection, lastDisconnect }) => {
        if (connection === 'close' && lastDisconnect?.error?.output?.statusCode !== DisconnectReason.loggedOut) {
            startBot();
        }
    });
};

startBot();
```
