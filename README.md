# discord-bot-
bot discord 
================================================================
FILE: bot/discord-bot/package.json
================================================================
{
  "name": "@workspace/discord-bot",
  "version": "0.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "tsx watch src/index.ts",
    "start": "tsx src/index.ts",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "discord.js": "^14.16.3",
    "openai": "^4.73.1"
  },
  "devDependencies": {
    "@types/node": "catalog:",
    "tsx": "catalog:",
    "typescript": "~5.9.2"
  }
}


================================================================
FILE: bot/discord-bot/tsconfig.json
================================================================
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "target": "ES2022",
    "outDir": "./dist",
    "rootDir": "./src",
    "noEmit": true,
    "skipLibCheck": true,
    "strict": true,
    "esModuleInterop": true,
    "resolveJsonModule": true
  },
  "include": ["src/**/*"]
}


================================================================
FILE: bot/discord-bot/src/ai.ts
================================================================
import OpenAI from "openai";

const baseURL = process.env.AI_INTEGRATIONS_OPENAI_BASE_URL;
const apiKey = process.env.AI_INTEGRATIONS_OPENAI_API_KEY;

if (!baseURL || !apiKey) {
  throw new Error(
    "Missing AI_INTEGRATIONS_OPENAI_BASE_URL or AI_INTEGRATIONS_OPENAI_API_KEY",
  );
}

export const openai = new OpenAI({ baseURL, apiKey });

export type ChatPart =
  | { type: "text"; text: string }
  | { type: "image_url"; image_url: { url: string } };

export interface ChatMessage {
  role: "system" | "user" | "assistant";
  content: string | ChatPart[];
}

export async function chat(messages: ChatMessage[]): Promise<string> {
  const res = await openai.chat.completions.create({
    model: "gpt-5.4",
    max_completion_tokens: 8192,
    // eslint-disable-next-line @typescript-eslint/no-explicit-any
    messages: messages as any,
  });
  return res.choices[0]?.message?.content ?? "";
}

export async function chatJSON<T>(messages: ChatMessage[]): Promise<T> {
  const res = await openai.chat.completions.create({
    model: "gpt-5.4",
    max_completion_tokens: 8192,
    response_format: { type: "json_object" },
    // eslint-disable-next-line @typescript-eslint/no-explicit-any
    messages: messages as any,
  });
  const text = res.choices[0]?.message?.content ?? "{}";
  return JSON.parse(text) as T;
}


================================================================
FILE: bot/discord-bot/src/storage.ts
================================================================
import { promises as fs } from "node:fs";
import path from "node:path";

const DATA_DIR = path.resolve(process.cwd(), "bot/discord-bot/data");
const COMMANDS_FILE = path.join(DATA_DIR, "commands.json");
const TICKETS_FILE = path.join(DATA_DIR, "tickets.json");

export interface DynamicCommand {
  name: string;
  description: string;
  responseTemplate: string;
  embed?: {
    title?: string;
    description?: string;
    color?: number;
  };
  buttons?: Array<{
    label: string;
    style: "primary" | "secondary" | "success" | "danger" | "link";
    url?: string;
    customId?: string;
    replyText?: string;
  }>;
  createdBy: string;
  createdAt: number;
}

export interface TicketInfo {
  channelId: string;
  guildId: string;
  userId: string;
  openedAt: number;
}

async function ensureDir() {
  await fs.mkdir(DATA_DIR, { recursive: true });
}

async function readJson<T>(file: string, fallback: T): Promise<T> {
  try {
    const txt = await fs.readFile(file, "utf-8");
    return JSON.parse(txt) as T;
  } catch {
    return fallback;
  }
}

async function writeJson(file: string, data: unknown): Promise<void> {
  await ensureDir();
  await fs.writeFile(file, JSON.stringify(data, null, 2), "utf-8");
}

export async function loadCommands(): Promise<DynamicCommand[]> {
  return readJson<DynamicCommand[]>(COMMANDS_FILE, []);
}

export async function saveCommands(cmds: DynamicCommand[]): Promise<void> {
  await writeJson(COMMANDS_FILE, cmds);
}

export async function addCommand(cmd: DynamicCommand): Promise<DynamicCommand[]> {
  const cmds = await loadCommands();
  const filtered = cmds.filter((c) => c.name !== cmd.name);
  filtered.push(cmd);
  await saveCommands(filtered);
  return filtered;
}

export async function removeCommand(name: string): Promise<DynamicCommand[]> {
  const cmds = await loadCommands();
  const filtered = cmds.filter((c) => c.name !== name);
  await saveCommands(filtered);
  return filtered;
}

export async function loadTickets(): Promise<Record<string, TicketInfo>> {
  return readJson<Record<string, TicketInfo>>(TICKETS_FILE, {});
}

export async function saveTicket(info: TicketInfo): Promise<void> {
  const tickets = await loadTickets();
  tickets[info.channelId] = info;
  await writeJson(TICKETS_FILE, tickets);
}

export async function deleteTicket(channelId: string): Promise<void> {
  const tickets = await loadTickets();
  delete tickets[channelId];
  await writeJson(TICKETS_FILE, tickets);
}

export async function isTicketChannel(channelId: string): Promise<boolean> {
  const tickets = await loadTickets();
  return Boolean(tickets[channelId]);
}


================================================================
FILE: bot/discord-bot/src/commands.ts
================================================================
import {
  REST,
  Routes,
  SlashCommandBuilder,
  type RESTPostAPIChatInputApplicationCommandsJSONBody,
} from "discord.js";
import { loadCommands } from "./storage.js";

export const builtInCommands: RESTPostAPIChatInputApplicationCommandsJSONBody[] = [
  new SlashCommandBuilder()
    .setName("panelticket")
    .setDescription("Post a ticket panel in this channel")
    .addStringOption((o) =>
      o
        .setName("title")
        .setDescription("Panel title")
        .setRequired(false),
    )
    .addStringOption((o) =>
      o
        .setName("description")
        .setDescription("Panel description")
        .setRequired(false),
    )
    .toJSON(),
  new SlashCommandBuilder()
    .setName("ping")
    .setDescription("Check the bot is alive")
    .toJSON(),
  new SlashCommandBuilder()
    .setName("createcommand")
    .setDescription("Ask the AI to create a new slash command for you")
    .addStringOption((o) =>
      o
        .setName("instruction")
        .setDescription("Describe the command you want")
        .setRequired(true),
    )
    .toJSON(),
  new SlashCommandBuilder()
    .setName("removecommand")
    .setDescription("Remove a previously-created custom command")
    .addStringOption((o) =>
      o
        .setName("name")
        .setDescription("Name of the command to remove")
        .setRequired(true),
    )
    .toJSON(),
  new SlashCommandBuilder()
    .setName("listcommands")
    .setDescription("List custom commands created via the AI")
    .toJSON(),
  new SlashCommandBuilder()
    .setName("close")
    .setDescription("Close the current ticket")
    .toJSON(),
];

export async function registerAllCommands(
  token: string,
  clientId: string,
): Promise<void> {
  const rest = new REST({ version: "10" }).setToken(token);
  const dynamic = await loadCommands();

  const dynamicJson = dynamic.map((c) =>
    new SlashCommandBuilder()
      .setName(c.name)
      .setDescription(c.description.slice(0, 100) || "Custom command")
      .toJSON(),
  );

  const body = [...builtInCommands, ...dynamicJson];
  await rest.put(Routes.applicationCommands(clientId), { body });
}


================================================================
FILE: bot/discord-bot/src/handlers/panelticket.ts
================================================================
import {
  ActionRowBuilder,
  ButtonBuilder,
  ButtonStyle,
  ChannelType,
  EmbedBuilder,
  PermissionFlagsBits,
  type ButtonInteraction,
  type ChatInputCommandInteraction,
  type Guild,
} from "discord.js";
import { saveTicket, deleteTicket, isTicketChannel } from "../storage.js";

export async function handlePanelTicket(
  interaction: ChatInputCommandInteraction,
) {
  const title =
    interaction.options.getString("title") ?? "Support Tickets";
  const description =
    interaction.options.getString("description") ??
    "Need help? Click the button below to open a private ticket. Our team will be with you shortly.";

  const embed = new EmbedBuilder()
    .setTitle(title)
    .setDescription(description)
    .setColor(0x5865f2);

  const row = new ActionRowBuilder<ButtonBuilder>().addComponents(
    new ButtonBuilder()
      .setCustomId("ticket:open")
      .setLabel("Open Ticket")
      .setStyle(ButtonStyle.Primary)
      .setEmoji("🎫"),
  );

  await interaction.reply({ embeds: [embed], components: [row] });
}

export async function handleOpenTicket(interaction: ButtonInteraction) {
  if (!interaction.guild) {
    await interaction.reply({
      content: "Tickets can only be opened in a server.",
      ephemeral: true,
    });
    return;
  }

  await interaction.deferReply({ ephemeral: true });

  const guild = interaction.guild;
  const user = interaction.user;

  const safeName = user.username
    .toLowerCase()
    .replace(/[^a-z0-9-]/g, "")
    .slice(0, 20) || "user";

  const channel = await guild.channels.create({
    name: `ticket-${safeName}`,
    type: ChannelType.GuildText,
    topic: `Ticket opened by ${user.tag} (${user.id})`,
    permissionOverwrites: [
      {
        id: guild.roles.everyone.id,
        deny: [PermissionFlagsBits.ViewChannel],
      },
      {
        id: user.id,
        allow: [
          PermissionFlagsBits.ViewChannel,
          PermissionFlagsBits.SendMessages,
          PermissionFlagsBits.AttachFiles,
          PermissionFlagsBits.ReadMessageHistory,
        ],
      },
      {
        id: interaction.client.user.id,
        allow: [
          PermissionFlagsBits.ViewChannel,
          PermissionFlagsBits.SendMessages,
          PermissionFlagsBits.ManageChannels,
          PermissionFlagsBits.ReadMessageHistory,
          PermissionFlagsBits.AttachFiles,
          PermissionFlagsBits.EmbedLinks,
        ],
      },
    ],
  });

  await saveTicket({
    channelId: channel.id,
    guildId: guild.id,
    userId: user.id,
    openedAt: Date.now(),
  });

  const welcome = new EmbedBuilder()
    .setTitle(`Welcome, ${user.username}!`)
    .setDescription(
      "Tell us what you need help with. You can write in any language, and feel free to attach screenshots or files. The assistant will reply automatically while a staff member is on the way.",
    )
    .setColor(0x57f287);

  const closeRow = new ActionRowBuilder<ButtonBuilder>().addComponents(
    new ButtonBuilder()
      .setCustomId("ticket:close")
      .setLabel("Close Ticket")
      .setStyle(ButtonStyle.Danger)
      .setEmoji("🔒"),
  );

  await channel.send({
    content: `<@${user.id}>`,
    embeds: [welcome],
    components: [closeRow],
  });

  await interaction.editReply({
    content: `Your ticket has been opened: <#${channel.id}>`,
  });
}

export async function handleCloseTicket(
  interaction: ButtonInteraction | ChatInputCommandInteraction,
) {
  if (!interaction.guild || !interaction.channel) {
    await interaction.reply({
      content: "This can only be used inside a ticket channel.",
      ephemeral: true,
    });
    return;
  }

  const channelId = interaction.channel.id;
  if (!(await isTicketChannel(channelId))) {
    await interaction.reply({
      content: "This is not an active ticket channel.",
      ephemeral: true,
    });
    return;
  }

  await interaction.reply({
    content: "Closing this ticket in 5 seconds...",
  });

  setTimeout(async () => {
    try {
      await deleteTicket(channelId);
      const channel = (interaction.guild as Guild).channels.cache.get(channelId);
      if (channel?.isTextBased() && "delete" in channel) {
        await channel.delete("Ticket closed");
      }
    } catch (err) {
      console.error("Failed to close ticket:", err);
    }
  }, 5000);
}


================================================================
FILE: bot/discord-bot/src/handlers/createcommand.ts
================================================================
import {
  ChatInputCommandInteraction,
  EmbedBuilder,
} from "discord.js";
import { chatJSON } from "../ai.js";
import {
  addCommand,
  loadCommands,
  removeCommand,
  type DynamicCommand,
} from "../storage.js";
import { registerAllCommands } from "../commands.js";

interface AICommandSpec {
  name: string;
  description: string;
  responseTemplate: string;
  embed?: {
    title?: string;
    description?: string;
    color?: number;
  };
}

export async function handleCreateCommand(
  interaction: ChatInputCommandInteraction,
  token: string,
  clientId: string,
) {
  const instruction = interaction.options.getString("instruction", true);
  await interaction.deferReply({ ephemeral: true });

  let spec: AICommandSpec;
  try {
    spec = await chatJSON<AICommandSpec>([
      {
        role: "system",
        content:
          "You design Discord slash commands. Reply ONLY with JSON matching this shape: " +
          '{"name":"lowercase-name","description":"short description (max 100 chars)","responseTemplate":"the message the bot will send when the command is used","embed":{"title":"optional","description":"optional","color":0xRRGGBB as integer or omit}}. ' +
          "Rules: name must be 1-32 chars, lowercase letters, numbers and dashes only. responseTemplate is plain text. If the user wants an embed, include the embed object with at least description. Never include code execution or external API calls. The command can only post a fixed embed/text. Reply in the same language the user used for content fields, but keep `name` ascii.",
      },
      { role: "user", content: instruction },
    ]);
  } catch (err) {
    await interaction.editReply({
      content: `I couldn't understand that. Try rephrasing. (${(err as Error).message})`,
    });
    return;
  }

  if (!spec.name || !/^[a-z0-9-]{1,32}$/.test(spec.name)) {
    await interaction.editReply({
      content: `Invalid command name produced (\`${spec.name}\`). Try rephrasing.`,
    });
    return;
  }

  const reserved = new Set([
    "panelticket",
    "ping",
    "createcommand",
    "removecommand",
    "listcommands",
    "close",
  ]);
  if (reserved.has(spec.name)) {
    await interaction.editReply({
      content: `\`${spec.name}\` is a built-in command and cannot be overridden.`,
    });
    return;
  }

  const cmd: DynamicCommand = {
    name: spec.name,
    description: (spec.description || "Custom command").slice(0, 100),
    responseTemplate: spec.responseTemplate || "",
    embed: spec.embed,
    createdBy: interaction.user.id,
    createdAt: Date.now(),
  };

  await addCommand(cmd);
  await registerAllCommands(token, clientId);

  const preview = new EmbedBuilder()
    .setTitle(`Created /${cmd.name}`)
    .setDescription(cmd.description)
    .setColor(0x57f287);
  if (cmd.responseTemplate) {
    preview.addFields({
      name: "Response",
      value: cmd.responseTemplate.slice(0, 1000),
    });
  }

  await interaction.editReply({
    content:
      "Command created. It may take a few seconds to appear in the slash menu.",
    embeds: [preview],
  });
}

export async function handleRemoveCommand(
  interaction: ChatInputCommandInteraction,
  token: string,
  clientId: string,
) {
  const name = interaction.options.getString("name", true).toLowerCase();
  await interaction.deferReply({ ephemeral: true });

  const before = await loadCommands();
  if (!before.find((c) => c.name === name)) {
    await interaction.editReply({
      content: `No custom command named \`${name}\`.`,
    });
    return;
  }

  await removeCommand(name);
  await registerAllCommands(token, clientId);

  await interaction.editReply({
    content: `Removed /${name}.`,
  });
}

export async function handleListCommands(
  interaction: ChatInputCommandInteraction,
) {
  const cmds = await loadCommands();
  if (cmds.length === 0) {
    await interaction.reply({
      content: "No custom commands yet. Use `/createcommand` to make one.",
      ephemeral: true,
    });
    return;
  }

  const embed = new EmbedBuilder()
    .setTitle("Custom Commands")
    .setColor(0x5865f2)
    .setDescription(
      cmds
        .map((c) => `**/${c.name}** — ${c.description}`)
        .join("\n")
        .slice(0, 4000),
    );

  await interaction.reply({ embeds: [embed], ephemeral: true });
}

export async function runDynamicCommand(
  interaction: ChatInputCommandInteraction,
) {
  const cmds = await loadCommands();
  const cmd = cmds.find((c) => c.name === interaction.commandName);
  if (!cmd) return false;

  const embeds: EmbedBuilder[] = [];
  if (cmd.embed && (cmd.embed.title || cmd.embed.description)) {
    const e = new EmbedBuilder();
    if (cmd.embed.title) e.setTitle(cmd.embed.title);
    if (cmd.embed.description) e.setDescription(cmd.embed.description);
    if (typeof cmd.embed.color === "number") e.setColor(cmd.embed.color);
    embeds.push(e);
  }

  await interaction.reply({
    content: cmd.responseTemplate || (embeds.length ? undefined : "."),
    embeds: embeds.length ? embeds : undefined,
  });
  return true;
}


================================================================
FILE: bot/discord-bot/src/handlers/aiReply.ts
================================================================
import {
  ChannelType,
  type Message,
  type TextChannel,
} from "discord.js";
import { chat, type ChatMessage, type ChatPart } from "../ai.js";
import { isTicketChannel } from "../storage.js";

const SYSTEM_PROMPT = `You are a helpful Discord support assistant inside a ticket channel.
- Always reply in the same language the user wrote in. You speak every language.
- Be friendly, concise, and useful. Ask one clarifying question at a time when needed.
- If the user attaches an image, describe what you see and respond to it directly.
- If the user attaches a text/code/log file (provided as text below), read it and help.
- If the request needs a human, say so and reassure them a staff member will join.
- Never invent private user data. Never mention you are an AI unless asked.`;

const IMAGE_EXTS = [".png", ".jpg", ".jpeg", ".gif", ".webp"];
const TEXT_EXTS = [
  ".txt",
  ".log",
  ".json",
  ".js",
  ".ts",
  ".tsx",
  ".jsx",
  ".py",
  ".md",
  ".csv",
  ".html",
  ".css",
  ".yml",
  ".yaml",
  ".env",
  ".sh",
  ".sql",
];
const VIDEO_EXTS = [".mp4", ".mov", ".webm", ".mkv", ".avi"];

function ext(name: string): string {
  const i = name.lastIndexOf(".");
  return i === -1 ? "" : name.slice(i).toLowerCase();
}

async function fetchTextAttachment(url: string, max = 8000): Promise<string> {
  try {
    const res = await fetch(url);
    const txt = await res.text();
    return txt.length > max ? txt.slice(0, max) + "\n...[truncated]" : txt;
  } catch {
    return "";
  }
}

async function buildHistory(channel: TextChannel, max = 8): Promise<ChatMessage[]> {
  const fetched = await channel.messages.fetch({ limit: max });
  const sorted = [...fetched.values()].sort(
    (a, b) => a.createdTimestamp - b.createdTimestamp,
  );

  const out: ChatMessage[] = [];
  for (const m of sorted) {
    if (m.author.bot && m.author.id !== channel.client.user.id) continue;

    const parts: ChatPart[] = [];
    let textBody = m.content || "";

    for (const att of m.attachments.values()) {
      const e = ext(att.name ?? "");
      if (IMAGE_EXTS.includes(e)) {
        parts.push({ type: "image_url", image_url: { url: att.url } });
      } else if (TEXT_EXTS.includes(e)) {
        const body = await fetchTextAttachment(att.url);
        textBody += `\n\n[attached file ${att.name}]\n${body}`;
      } else if (VIDEO_EXTS.includes(e)) {
        textBody += `\n\n[user attached a video: ${att.name} (${att.size} bytes). I cannot watch the video, ask the user to describe what happens in it.]`;
      } else {
        textBody += `\n\n[user attached file: ${att.name} (${att.size} bytes)]`;
      }
    }

    if (textBody.trim()) {
      parts.unshift({ type: "text", text: textBody.trim() });
    }
    if (parts.length === 0) continue;

    out.push({
      role: m.author.id === channel.client.user.id ? "assistant" : "user",
      content: parts.length === 1 && parts[0].type === "text" ? parts[0].text : parts,
    });
  }
  return out;
}

export async function handleTicketMessage(message: Message): Promise<void> {
  if (message.author.bot) return;
  if (!message.guild) return;
  if (message.channel.type !== ChannelType.GuildText) return;
  if (!(await isTicketChannel(message.channel.id))) return;

  const channel = message.channel as TextChannel;

  try {
    await channel.sendTyping();
    const history = await buildHistory(channel);
    const messages: ChatMessage[] = [
      { role: "system", content: SYSTEM_PROMPT },
      ...history,
    ];
    const reply = await chat(messages);
    if (reply.trim()) {
      await channel.send(reply.slice(0, 1900));
    }
  } catch (err) {
    console.error("AI reply failed:", err);
    await channel.send(
      "I had trouble responding just now. A staff member will follow up.",
    );
  }
}


================================================================
FILE: bot/discord-bot/src/handlers/dmCreate.ts
================================================================
import { ChannelType, type Message } from "discord.js";
import { chatJSON } from "../ai.js";
import {
  addCommand,
  removeCommand,
  loadCommands,
  type DynamicCommand,
} from "../storage.js";
import { registerAllCommands } from "../commands.js";

interface AIIntent {
  action: "create" | "remove" | "list" | "chat";
  name?: string;
  description?: string;
  responseTemplate?: string;
  embed?: { title?: string; description?: string; color?: number };
  reply?: string;
}

const SYSTEM = `You manage a user's Discord bot via natural language. Decide what they want.
Reply ONLY with JSON in this shape:
{"action":"create"|"remove"|"list"|"chat","name":"lowercase-name","description":"short","responseTemplate":"text the bot will post","embed":{"title":"opt","description":"opt","color":int},"reply":"plain message back to the user when action is chat or after create/remove"}
Rules:
- "create": user wants a new slash command. name must be 1-32 chars, lowercase a-z 0-9 -. Pick a good name from the request. Set responseTemplate to what the command should reply with. Add embed only if it makes sense.
- "remove": user wants to delete a command. Provide the name.
- "list": user wants to see existing custom commands.
- "chat": user is just talking, answering, or asking a question. Put your friendly reply in "reply". Speak the same language they used.
- Always include a "reply" field that the bot will send back to the user describing what was done.
- Never refuse to make a command. If the user describes one, create it. The command can only post a fixed text/embed - that is fine.`;

export async function handleOwnerDM(
  message: Message,
  ownerId: string,
  token: string,
  clientId: string,
): Promise<void> {
  if (message.author.bot) return;
  if (message.channel.type !== ChannelType.DM) return;
  if (message.author.id !== ownerId) {
    await message.reply(
      "Only the bot owner can use DMs to manage commands.",
    );
    return;
  }

  const text = message.content.trim();
  if (!text) return;

  try {
    await message.channel.sendTyping();
    const intent = await chatJSON<AIIntent>([
      { role: "system", content: SYSTEM },
      { role: "user", content: text },
    ]);

    if (intent.action === "list") {
      const cmds = await loadCommands();
      if (cmds.length === 0) {
        await message.reply("No custom commands yet.");
        return;
      }
      await message.reply(
        "Custom commands:\n" +
          cmds.map((c) => `• /${c.name} — ${c.description}`).join("\n"),
      );
      return;
    }

    if (intent.action === "remove" && intent.name) {
      await removeCommand(intent.name);
      await registerAllCommands(token, clientId);
      await message.reply(intent.reply || `Removed /${intent.name}.`);
      return;
    }

    if (intent.action === "create" && intent.name) {
      if (!/^[a-z0-9-]{1,32}$/.test(intent.name)) {
        await message.reply(
          `That name (\`${intent.name}\`) is invalid. Try again.`,
        );
        return;
      }
      const cmd: DynamicCommand = {
        name: intent.name,
        description: (intent.description || "Custom command").slice(0, 100),
        responseTemplate: intent.responseTemplate || "",
        embed: intent.embed,
        createdBy: message.author.id,
        createdAt: Date.now(),
      };
      await addCommand(cmd);
      await registerAllCommands(token, clientId);
      await message.reply(
        intent.reply ||
          `Created /${cmd.name}. It may take a few seconds to appear.`,
      );
      return;
    }

    await message.reply(intent.reply || "Got it.");
  } catch (err) {
    console.error("Owner DM error:", err);
    await message.reply(
      `Something went wrong: ${(err as Error).message}`,
    );
  }
}


================================================================
FILE: bot/discord-bot/src/index.ts
================================================================
import {
  Client,
  Events,
  GatewayIntentBits,
  Partials,
  type Interaction,
} from "discord.js";
import { registerAllCommands } from "./commands.js";
import {
  handleCloseTicket,
  handleOpenTicket,
  handlePanelTicket,
} from "./handlers/panelticket.js";
import {
  handleCreateCommand,
  handleListCommands,
  handleRemoveCommand,
  runDynamicCommand,
} from "./handlers/createcommand.js";
import { handleTicketMessage } from "./handlers/aiReply.js";
import { handleOwnerDM } from "./handlers/dmCreate.js";

const TOKEN = process.env.DISCORD_BOT_TOKEN;
if (!TOKEN) {
  console.error("DISCORD_BOT_TOKEN is missing.");
  process.exit(1);
}

const OWNER_ID = process.env.DISCORD_OWNER_ID ?? "";

const client = new Client({
  intents: [
    GatewayIntentBits.Guilds,
    GatewayIntentBits.GuildMessages,
    GatewayIntentBits.MessageContent,
    GatewayIntentBits.DirectMessages,
  ],
  partials: [Partials.Channel, Partials.Message],
});

client.once(Events.ClientReady, async (c) => {
  console.log(`Bot ready as ${c.user.tag}`);
  try {
    await registerAllCommands(TOKEN, c.user.id);
    console.log("Slash commands registered.");
  } catch (err) {
    console.error("Failed to register commands:", err);
  }
});

client.on(Events.InteractionCreate, async (interaction: Interaction) => {
  try {
    if (interaction.isButton()) {
      if (interaction.customId === "ticket:open") {
        await handleOpenTicket(interaction);
      } else if (interaction.customId === "ticket:close") {
        await handleCloseTicket(interaction);
      }
      return;
    }

    if (!interaction.isChatInputCommand()) return;
    const clientId = interaction.client.user.id;

    switch (interaction.commandName) {
      case "ping":
        await interaction.reply({
          content: `Pong! ${interaction.client.ws.ping}ms`,
          ephemeral: true,
        });
        return;
      case "panelticket":
        await handlePanelTicket(interaction);
        return;
      case "createcommand":
        await handleCreateCommand(interaction, TOKEN, clientId);
        return;
      case "removecommand":
        await handleRemoveCommand(interaction, TOKEN, clientId);
        return;
      case "listcommands":
        await handleListCommands(interaction);
        return;
      case "close":
        await handleCloseTicket(interaction);
        return;
      default: {
        const handled = await runDynamicCommand(interaction);
        if (!handled) {
          await interaction.reply({
            content: "Unknown command.",
            ephemeral: true,
          });
        }
      }
    }
  } catch (err) {
    console.error("Interaction error:", err);
    if (interaction.isRepliable() && !interaction.replied) {
      try {
        await interaction.reply({
          content: "Something went wrong handling that.",
          ephemeral: true,
        });
      } catch {
        // ignore
      }
    }
  }
});

client.on(Events.MessageCreate, async (message) => {
  try {
    if (OWNER_ID && message.channel.type === 1 /* DM */) {
      await handleOwnerDM(message, OWNER_ID, TOKEN, client.user!.id);
      return;
    }
    await handleTicketMessage(message);
  } catch (err) {
    console.error("Message handler error:", err);
  }
});

client.login(TOKEN).catch((err) => {
  console.error("Failed to login:", err);
  process.exit(1);
});
