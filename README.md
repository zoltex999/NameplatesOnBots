# Discord Bot Nameplates — Complete Guide

Discord's Display Name Styles (internally known as Nameplates) allow you to customize how a bot's display name appears inside a specific server. This includes applying custom typography, color palettes, and special visual treatments like neon glows or gradient fades.

While human accounts must purchase these cosmetics from the Discord Shop or subscribe to Nitro, bots can use them natively for free. Because Discord treats these styles as guild-level member metadata rather than global inventory assets, the backend skips the typical entitlement checks when modifying a bot's server profile.

---

## How It Works Under the Hood

To change a nameplate, your bot issues an HTTP PATCH request to the current guild member profile endpoint:

```text
PATCH /guilds/{guild_id}/members/@me

```

The JSON payload accepts three dedicated text-styling parameters:

| Parameter Field | Type | Description |
| --- | --- | --- |
| display_name_font_id | Integer | The unique ID of the typography style to apply. |
| display_name_effect_id | Integer | The unique ID of the visual filter (Glow, Neon, etc.). |
| display_name_colors | Array (Int) | An array containing 1 or 2 base-10 decimal color values. |

Note that this configuration is strictly per-guild. Modifying the bot's nameplate in Server A will not affect its look globally or in Server B.

---

## Supported Options and IDs

### 1. Font Styles

| Font Name | ID | Visual Aesthetic |
| --- | --- | --- |
| **Default** | 11 | Standard Discord gg sans typography |
| **Tempo** | 1 | Bold, fast-paced athletic styling |
| **Sakura** | 3 | Soft, elegant brush script |
| **Jellybean** | 4 | Playful, rounded comic style |
| **Modern** | 6 | Clean, geometric sans-serif |
| **Medieval** | 7 | Gothic/Fraktur blackletter calligraphy |
| **8bit** | 8 | Retro pixelated arcade lettering |
| **Vampyre** | 10 | Sharp, aggressive display font |

### 2. Text Effects

| Effect Name | ID | Color Requirement |
| --- | --- | --- |
| **Solid** | 1 | Requires exactly 1 color |
| **Gradient** | 2 | Requires exactly 2 colors (for blending) |
| **Neon** | 3 | Requires exactly 1 color |
| **Toon** | 4 | Requires exactly 1 color |
| **Pop** | 5 | Requires exactly 1 color |
| **Glow** | 6 | Requires exactly 1 color |

---

## Color Conversion Utility

Discord's API will reject standard hexadecimal color strings (such as #FF69B4). Hex strings must be parsed into base-10 decimal integers before sending the network payload.

### JavaScript Helper

```javascript
function hexToDecimal(hex) {
    return parseInt(hex.replace('#', ''), 16);
}

const color = hexToDecimal('#FF69B4'); // Returns: 16738740

```

### Python Helper

```python
def hex_to_decimal(hex_str):
    return int(hex_str.lstrip('#'), 16)

color = hex_to_decimal('#FF69B4') // Returns: 16738740

```

---

## Production Code Examples

### 1. Discord.js v14 (Slash Command)

This command applies the Sakura font combined with a pink Glow effect.

```javascript
const { SlashCommandBuilder, Routes } = require('discord.js');

module.exports = {
    data: new SlashCommandBuilder()
        .setName('setbotstyle')
        .setDescription("Updates the bot's text styling in this server"),

    async execute(interaction) {
        await interaction.deferReply({ ephemeral: true });

        try {
            await interaction.client.rest.patch(
                Routes.guildMember(interaction.guildId, '@me'),
                {
                    body: {
                        display_name_font_id: 3,         // Sakura Font
                        display_name_effect_id: 6,       // Glow Effect
                        display_name_colors: [16738740]  // #FF69B4 in Decimal
                    }
                }
            );
            await interaction.editReply('Bot nameplate styling applied successfully.');
        } catch (error) {
            console.error(error);
            await interaction.editReply('Failed to update nameplate. Verify bot permissions.');
        }
    }
};

```

### 2. Discord.py (Prefix Command)

This command applies the Vampyre font combined with a dual-color Gradient effect.

```python
import discord
from discord.ext import commands

class NameplateStyle(commands.Cog):
    def __init__(self, bot: commands.Bot):
        self.bot = bot

    @commands.command(name="vampyrestyle")
    async def vampyre_style(self, ctx: commands.Context):
        # Build raw HTTP route mapping
        route = discord.http.Route('PATCH', f'/guilds/{ctx.guild.id}/members/@me')
        
        color_1 = int("FF2E63", 16)  # Electric Pink-Red
        color_2 = int("8B17E4", 16)  # Deep Purple

        payload = {
            "display_name_font_id": 10,                // Vampyre Font
            "display_name_effect_id": 2,               // Gradient Effect
            "display_name_colors": [color_1, color_2]  // Gradient requires exactly 2 colors
        }

        try:
            await self.bot.http.request(route, json=payload)
            await ctx.send("Vampyre gradient style successfully applied.")
        except discord.Forbidden:
            await ctx.send("Permission Denied: Ensure the bot has Change Nickname or correct role hierarchy.")
        except Exception as e:
            await ctx.send(f"API Error: {e}")

async def setup(bot: commands.Bot):
    await bot.add_cog(NameplateStyle(bot))

```

---

## Multi-Server Global Synchronization

To apply a consistent theme across every guild your bot resides in, iterate through the bot's guild cache using a loop:

```javascript
const payload = {
    display_name_font_id: 10,
    display_name_effect_id: 2,
    display_name_colors: [16723043, 9115620]
};

for (const guild of client.guilds.cache.values()) {
    try {
        await client.rest.patch(Routes.guildMember(guild.id, '@me'), { body: payload });
    } catch (err) {
        // Suppresses errors for individual servers where permissions are missing
        console.warn(`Could not sync nameplate in guild: ${guild.id}`);
    }
}

```

---

## Removing a Nameplate Style

To strip all text cosmetics and return the bot back to the default server appearance, pass 0 to the ID values and an empty array to the color field:

```json
{
  "display_name_font_id": 0,
  "display_name_effect_id": 0,
  "display_name_colors": []
}

```

---

## Troubleshooting and API Errors

* **400 Bad Request**: Color count mismatch or invalid ID. If using Gradient (ID 2), you must supply exactly 2 colors in the display_name_colors array. All other effects require exactly 1 color.
* **403 Forbidden**: Hierarchy or structural permission issue. Ensure the bot possesses the Change Nickname or Manage Nicknames permission. The bot's primary role must also be positioned above any roles it is trying to execute actions near.
* **No visual changes on client**: Client-side cache or visibility settings are hiding standard text. Navigate to your personal Discord client settings under User Settings > Accessibility and verify that Display Name Styles is checked to true.
