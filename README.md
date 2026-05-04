# discord-bot-
bot discord 
const { Client, GatewayIntentBits, REST, Routes, SlashCommandBuilder, EmbedBuilder, ActionRowBuilder, ButtonBuilder, ButtonStyle, PermissionFlagsBits, ChannelType } = require('discord.js');

const TOKEN = process.env.DISCORD_BOT_TOKEN;
const client = new Client({
  intents: [
    GatewayIntentBits.Guilds,
    GatewayIntentBits.GuildMessages,
    GatewayIntentBits.GuildMembers,
  ]
});

const commands = [
  new SlashCommandBuilder()
    .setName('ticketpanel')
    .setDescription('Send a ticket panel for users to open support tickets')
    .setDefaultMemberPermissions(PermissionFlagsBits.ManageChannels)
    .toJSON(),

  new SlashCommandBuilder()
    .setName('ban')
    .setDescription('Ban a member from the server')
    .addUserOption(opt => opt.setName('user').setDescription('The user to ban').setRequired(true))
    .addStringOption(opt => opt.setName('reason').setDescription('Reason for the ban').setRequired(false))
    .setDefaultMemberPermissions(PermissionFlagsBits.BanMembers)
    .toJSON(),

  new SlashCommandBuilder()
    .setName('kick')
    .setDescription('Kick a member from the server')
    .addUserOption(opt => opt.setName('user').setDescription('The user to kick').setRequired(true))
    .addStringOption(opt => opt.setName('reason').setDescription('Reason for the kick').setRequired(false))
    .setDefaultMemberPermissions(PermissionFlagsBits.KickMembers)
    .toJSON(),

  new SlashCommandBuilder()
    .setName('mute')
    .setDescription('Timeout (mute) a member')
    .addUserOption(opt => opt.setName('user').setDescription('The user to mute').setRequired(true))
    .addIntegerOption(opt => opt.setName('duration').setDescription('Duration in minutes').setRequired(true))
    .addStringOption(opt => opt.setName('reason').setDescription('Reason for the mute').setRequired(false))
    .setDefaultMemberPermissions(PermissionFlagsBits.ModerateMembers)
    .toJSON(),

  new SlashCommandBuilder()
    .setName('memberrole')
    .setDescription('Show member count for up to 5 roles')
    .addRoleOption(opt => opt.setName('role1').setDescription('First role').setRequired(true))
    .addRoleOption(opt => opt.setName('role2').setDescription('Second role').setRequired(false))
    .addRoleOption(opt => opt.setName('role3').setDescription('Third role').setRequired(false))
    .addRoleOption(opt => opt.setName('role4').setDescription('Fourth role').setRequired(false))
    .addRoleOption(opt => opt.setName('role5').setDescription('Fifth role').setRequired(false))
    .setDefaultMemberPermissions(PermissionFlagsBits.ManageRoles)
    .toJSON(),
];

client.once('clientReady', async () => {
  console.log(`Bot is online as ${client.user.tag}`);
  const rest = new REST({ version: '10' }).setToken(TOKEN);
  try {
    await rest.put(Routes.applicationCommands(client.user.id), { body: commands });
    console.log('Slash commands registered!');
  } catch (err) {
    console.error('Error registering commands:', err);
  }
});

client.on('interactionCreate', async (interaction) => {
  if (interaction.isChatInputCommand()) {

    // /ticketpanel
    if (interaction.commandName === 'ticketpanel') {
      const embed = new EmbedBuilder()
        .setTitle('ð« Support Tickets')
        .setDescription('Need help? Click the button below to open a support ticket.\nOur team will get back to you as soon as possible.')
        .setColor(0x5865F2)
        .setFooter({ text: 'Click the button to create a ticket' });
      const row = new ActionRowBuilder().addComponents(
        new ButtonBuilder().setCustomId('create_ticket').setLabel('Open a Ticket').setStyle(ButtonStyle.Primary).setEmoji('ð«')
      );
      await interaction.channel.send({ embeds: [embed], components: [row] });
      await interaction.reply({ content: 'Ticket panel sent!', ephemeral: true });
    }

    // /ban
    if (interaction.commandName === 'ban') {
      const target = interaction.options.getMember('user');
      const reason = interaction.options.getString('reason') || 'No reason provided';
      if (!target) return interaction.reply({ content: 'User not found.', ephemeral: true });
      if (!target.bannable) return interaction.reply({ content: 'I cannot ban this user.', ephemeral: true });
      await target.ban({ reason });
      await interaction.reply({ embeds: [new EmbedBuilder().setColor(0xFF0000).setTitle('ð¨ User Banned').addFields(
        { name: 'User', value: `${target.user.tag}`, inline: true },
        { name: 'Moderator', value: `${interaction.user.tag}`, inline: true },
        { name: 'Reason', value: reason }
      )]});
    }

    // /kick
    if (interaction.commandName === 'kick') {
      const target = interaction.options.getMember('user');
      const reason = interaction.options.getString('reason') || 'No reason provided';
      if (!target) return interaction.reply({ content: 'User not found.', ephemeral: true });
      if (!target.kickable) return interaction.reply({ content: 'I cannot kick this user.', ephemeral: true });
      await target.kick(reason);
      await interaction.reply({ embeds: [new EmbedBuilder().setColor(0xFF6600).setTitle('ð¢ User Kicked').addFields(
        { name: 'User', value: `${target.user.tag}`, inline: true },
        { name: 'Moderator', value: `${interaction.user.tag}`, inline: true },
        { name: 'Reason', value: reason }
      )]});
    }

    // /mute
    if (interaction.commandName === 'mute') {
      const target = interaction.options.getMember('user');
      const duration = interaction.options.getInteger('duration');
      const reason = interaction.options.getString('reason') || 'No reason provided';
      if (!target) return interaction.reply({ content: 'User not found.', ephemeral: true });
      if (!target.moderatable) return interaction.reply({ content: 'I cannot mute this user.', ephemeral: true });
      await target.timeout(duration * 60 * 1000, reason);
      await interaction.reply({ embeds: [new EmbedBuilder().setColor(0xFFFF00).setTitle('ð User Muted').addFields(
        { name: 'User', value: `${target.user.tag}`, inline: true },
        { name: 'Moderator', value: `${interaction.user.tag}`, inline: true },
        { name: 'Duration', value: `${duration} minute(s)`, inline: true },
        { name: 'Reason', value: reason }
      )]});
    }

    // /memberrole
    if (interaction.commandName === 'memberrole') {
      await interaction.deferReply();

      const roles = [];
      for (let i = 1; i <= 5; i++) {
        const role = interaction.options.getRole(`role${i}`);
        if (role) roles.push(role);
      }

      // Fetch all members to get accurate counts
      await interaction.guild.members.fetch();

      let rolesText = '';
      for (const role of roles) {
        const count = role.members.size;
        rolesText += `<@&${role.id}> â¢ ${count} members\n`;
      }

      const embed = new EmbedBuilder()
        .setTitle('ð Prime Notifier Status')
        .setColor(0x5865F2)
        .addFields(
          { name: 'ð Ping Roles', value: rolesText || 'No roles found' }
        )
        .setFooter({ text: 'Prime Notifier â¢ Member Stats' })
        .setTimestamp();

      await interaction.editReply({ embeds: [embed] });
    }
  }

  // Button: Create Ticket
  if (interaction.isButton() && interaction.customId === 'create_ticket') {
    const guild = interaction.guild;
    const user = interaction.user;
    const existing = guild.channels.cache.find(c => c.name === `ticket-${user.username.toLowerCase()}`);
    if (existing) return interaction.reply({ content: `You already have an open ticket: ${existing}`, ephemeral: true });

    const ticketChannel = await guild.channels.create({
      name: `ticket-${user.username.toLowerCase()}`,
      type: ChannelType.GuildText,
      permissionOverwrites: [
        { id: guild.id, deny: [PermissionFlagsBits.ViewChannel] },
        { id: user.id, allow: [PermissionFlagsBits.ViewChannel, PermissionFlagsBits.SendMessages, PermissionFlagsBits.ReadMessageHistory] },
      ],
    });

    const closeRow = new ActionRowBuilder().addComponents(
      new ButtonBuilder().setCustomId('close_ticket').setLabel('Close Ticket').setStyle(ButtonStyle.Danger).setEmoji('ð')
    );

    await ticketChannel.send({
      content: `<@${user.id}>`,
      embeds: [new EmbedBuilder().setColor(0x5865F2).setTitle('ð« Ticket Opened')
        .setDescription(`Hello ${user.username}! A staff member will be with you shortly.\nDescribe your issue below.`)
        .setFooter({ text: 'Click "Close Ticket" when your issue is resolved.' })],
      components: [closeRow]
    });
    await interaction.reply({ content: `Your ticket has been created: ${ticketChannel}`, ephemeral: true });
  }

  // Button: Close Ticket
  if (interaction.isButton() && interaction.customId === 'close_ticket') {
    await interaction.reply({ content: 'ð Closing ticket in 5 seconds...' });
    setTimeout(() => interaction.channel.delete().catch(() => {}), 5000);
  }
});

client.login(TOKEN);
