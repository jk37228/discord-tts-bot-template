const { Client, GatewayIntentBits, SlashCommandBuilder, REST, Routes, Events } = require('discord.js');
const { joinVoiceChannel, createAudioPlayer, createAudioResource, AudioPlayerStatus } = require('@discordjs/voice');
const googleTTS = require('google-tts-api');
const fs = require('fs');
const https = require('https');
const path = require('path');

const TOKEN = 'MTM4Nzg4MDU0MTk0NDAyMTEyMg.Gsbcrj.HtxF_OxxmP8C990IS-5T__WWVgnklkM73shqzg';
const CLIENT_ID = '1387880541944021122';
const GUILD_ID = '1374436123253801142';

const client = new Client({
  intents: [
    GatewayIntentBits.Guilds,
    GatewayIntentBits.GuildVoiceStates,
    GatewayIntentBits.GuildMessages,
    GatewayIntentBits.MessageContent,
    GatewayIntentBits.GuildMembers
  ]
});

let voiceConnection;
let audioPlayer = createAudioPlayer();
let currentVoiceChannelId = null;

client.once(Events.ClientReady, () => {
  console.log(`✅ Logged in as ${client.user.tag}`);
});

const commands = [
  new SlashCommandBuilder()
    .setName('join')
    .setDescription('Fai entrare il bot nella tua vocale')
].map(cmd => cmd.toJSON());

const rest = new REST({ version: '10' }).setToken(TOKEN);
(async () => {
  try {
    await rest.put(Routes.applicationGuildCommands(CLIENT_ID, GUILD_ID), { body: commands });
    console.log('✅ Comando /join registrato');
  } catch (error) {
    console.error(error);
  }
})();

client.on(Events.InteractionCreate, async interaction => {
  if (!interaction.isChatInputCommand()) return;
  if (interaction.commandName === 'join') {
    const member = interaction.member;
    const voiceChannel = member.voice.channel;
    if (!voiceChannel) return interaction.reply({ content: '⚠️ Devi essere in un canale vocale!', ephemeral: true });

    voiceConnection = joinVoiceChannel({
      channelId: voiceChannel.id,
      guildId: voiceChannel.guild.id,
      adapterCreator: voiceChannel.guild.voiceAdapterCreator
    });

    voiceConnection.subscribe(audioPlayer);
    currentVoiceChannelId = voiceChannel.id;
    await interaction.reply('✅ Sono entrato nella vocale!');
  }
});

client.on(Events.MessageCreate, async message => {
  if (message.author.bot || !voiceConnection || !message.guild) return;
  const voiceChannel = message.guild.channels.cache.get(currentVoiceChannelId);
  if (!voiceChannel || !voiceChannel.members.has(message.author.id)) return;

  const text = message.content.slice(0, 200);
  try {
    const url = googleTTS.getAudioUrl(text, { lang: 'it', slow: false, host: 'https://translate.google.com' });
    const filePath = path.join(__dirname, 'tts.mp3');
    const file = fs.createWriteStream(filePath);
    https.get(url, res => {
      res.pipe(file);
      file.on('finish', () => {
        file.close(() => {
          const resource = createAudioResource(filePath);
          audioPlayer.play(resource);
          audioPlayer.once(AudioPlayerStatus.Idle, () => fs.unlink(filePath, () => {}));
        });
      });
    });
  } catch (err) {
    console.error('Errore TTS:', err);
  }
});

client.login(TOKEN);

