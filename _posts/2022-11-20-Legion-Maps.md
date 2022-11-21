---
tags: Discord Legion
---

# Rivals Legion Maps

## Background
Rivals is a Command and Conquer game for mobile that came out a few years ago. I'm going to skip right past discussing that as a concept. It does have a tight-knit community of players. Rivals was the first game I really used Discord for interacting with other players. Thanks to Davo (another member of the Rivals community) it also became the first scenario I ended up writing a Discord bot for. This was early Discord bot days. So before intents, and before slash commands. 

The simplest use case for a bot for Rivals is to show a map. Rivals has 48 maps you can play on. And of ccourseyou can challenge other players to a game. So it makes sense to want to look up the map ahead of time. 

This is probably my favorite map (the only map I've beaten 13lade on, that's a story for another time):
![Orbit](http://davoonline.com/rivals/Bot/maps/orbit.png)

While Rivals has a random map feature in game it's revealed at the last moment when a match starts. For tournaments it helps to pick a map at random, but ahead of time so that both players know what map to expect and can pick thier decks accordingly. Finally, despite what looks like on the surface to be a small map and therefore simple, it turns out there is a lot of nuance to maps. Some are not at all fun to play on and poorly balanced. I'm looking at you:

![Canal Row](http://davoonline.com/rivals/Bot/maps/canalrow.png)

## Features
The use case is perfect for getting started with a bot. There's no calculation. And the simplest function, a command that just picks a map at random, and has no arguments. In the first days that would be a "\~map" command, that would go through the message received handler. Legion uses the "~" prefix to identify commands. This of course means for a simple use case this bot would need to read message content. But you could limit it to a specific channel at least. This was the typical model for those of us that were vigilant about security. 

I looked over the code yesterday, and I forgot just how much more there can be to it. I completely forgot that the bot support localization in Russian and German (translations provided to me by members of the community). As one small touch, for example, it responds to ~karte for German as well as ~map.

Beyond picking a map, players often want to see a particular map ahead of time. So you want something like "~map orbit". This immediately brings up the observation that bots start to make Discord look like a command like shell, but missing auto complete and tool tips. Discord's App Commands help address once they were introduced.  But until then the bot needed a way to get the map that was closest to what was type. Cause if you typed ~map \<name\> you clearly wanted a map, so it should always return the best guest at a map. 

Finally, for tournaments, or just for fun, you could create and store map groups. Its a named list of maps, that could also be used to supply the maps used for a tournament. 

## Edit Distance

For making sure the command always returned a map I started with https://www.nuget.org/packages/Fastenshtein. Given an input it would return the closest localized map name. One trick it would do is remove "the", or "das" and "der" if in German, from the map name as this would create some noise in the edit distance. Looking at the code now I found a small bug.  I used the wrong normalizer for one of the languages so the wrong words would get removed before looking up edit distance.

Because the command ALWAYS returns a map now based on edit distance, there was some brief fun around looking up what maps a particular player name would result in. For example, "Sarpedon" has the smallest edit distance of all the maps to "Caged In"

## Using a Slash Command
So now there are app commands, and "map" makes for a great global command. This is the snippet for adding the command: 

```C#
        SlashCommandProperties guildMessageCommand = new SlashCommandBuilder()
            .WithName(CommandName)
            .WithDescription("Picks a map at random")
            .WithDMPermission(true)
            .WithDefaultMemberPermissions(GuildPermission.SendMessages)
            .Build();

await client.BulkOverwriteGlobalApplicationCommandsAsync( new ApplicationCommandProperties[] {guildMessageCommand});
```
This is a snippet of the handler: 
```C#
    private Task _client_SlashCommandExecuted(SocketSlashCommand arg)
    {
        if (arg.CommandName == CommandName)
        {
            return arg.RespondAsync(embed: ShowMap(this.maps[R.Next(this.maps.Count)]).Build());
        }
        return Task.CompletedTask;
    }

    private static EmbedBuilder ShowMap(MapValues map)
    {
        EmbedBuilder builder = new EmbedBuilder();
        builder.WithTitle(MapLookup.GetMapName(map));
        builder.WithImageUrl(MapLookup.GetUrl(map));
        return builder;
    }
```

And using application commands makes it really easy to add this command to a server: 
https://discord.com/api/oauth2/authorize?client_id=1043771859159756840&scope=applications.commands

There is some boilerplate to a bot, however. For example:
* Logging
* Storing the secret

As part of creating this command I think I have a good template I can leverage in the future, and share.

## Options
I might be able to make it easier to look up a name by showing a list of all names as part of the slash command. Discord commands can be created with pre-set options for the arguments. Unfortunately the limit to options is 25. But there is grouping, so I can break down the 48 maps into some grouping (maybe alphabetically) so each group would have less than 25 options. 

Finally, there is an auto-completion capability as well: https://discord.com/developers/docs/interactions/application-commands#autocomplete. Lots of way to improve the simple command with Discord features. 

So that's a brief random walk through bots, maps, Legion, and Rivals. The intent is there will be a few more Legion related posts!

Visit 13lade's Twitch: [https://www.twitch.tv/13lade](https://www.twitch.tv/13lade)
There's a Rivals tournament facilitated by Legion today on this channel!

Please support Legion and other tech: <a href="https://www.buymeacoffee.com/sarpedontdw" target="_blank"><img src="https://cdn.buymeacoffee.com/buttons/default-orange.png" alt="Buy Me A Coffee" height="41" width="174"></a>
