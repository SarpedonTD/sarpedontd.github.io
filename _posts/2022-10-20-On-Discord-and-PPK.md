# On Discord and PPK

First, for anyone not familiar with the acronym, in EVE PPK is Pay Per Kill.

EVE runs on Google Sheets and Discord. I'll get more into Discord particulars later. But PPK systems are one reason. 
A central part of EVE Echoes is a kill mail. Here's one for reference:

## Overview
![A Kill Mail](https://cdn.discordapp.com/attachments/758102282276700232/908505170760056832/unknown.png)

The game, unlike EVE Online, does not have a KM API. Most alliances therefore have to implement some work around. And Discord is often central to that work around.
It starts with OCR of the KM above when shared. I won't go into that here, because that's not  a part I am familiar with. Though I thank the devs that do that portion. 
Suffice it say after OCR you have the various fields from the image available. Things like: 
* The victim and victim's corp. 
* The value. 
* The Final Blow and Top Damage participants. 

The typical flow for building an alliance PPK program goes like this: 

1. User grabs an image of the KM from the game, and posts an often cropped version of it into a Discord channel dedicated to kill mails. 
2. A bot picks up the image, runs it through OCR, and outputs the information parsed into some schematized store. 
3. Some analytics are run and output somewhere for alliance members to see.

## PPK Events
For VOID, I wanted to expand on what PPK means. Typically PPK is for final blow. The person gets some % of the kill in ISK back. 
So in VOID's system, I have a KM still, but also added a separate PPK event. A typical KM already has three PPK events: 
* Damage
* Final shot
* Participants. 

All of these we can give PPK for. On top of that, things that don't appear on the kill mail, but we can add as PPK events: 
* The FC (Fleet Commander)
* All Logi (EVE healers)
* Tackle, and interdiction
* Scouts

This will look like a DKP system for anyone familiar with DKP (Dragon Kill Points) from other MMOs. But the DKP is backed by ISK, which is obtained in-game.

## Pilot Workflow

The workflow, from a Pilot's perspective, looks like this: 

1. Get a KM
2. Post the KM to dedicated Discord channel
3. Other pilot's involved in the KM can then added their own PPK events. FCs, tackle, can add PPK events through commands or buttons createed by a Discord bot. 
Because its in a visible channel there's very little risk of someone lying. 
4. A summary is created run at some point, and through some manual method the pilots are given the PPK for the week/month or for each individual PPK event through a ticket bot.

## Storage for the Workflow

For design of storage, most systems have an OLTP and OLAP system. 
For VOID Azure Table backs the initial ingress of KM and PPK events. I'll leave the schema for a different time. 
And we creatively use Google Sheets as the OLAP store. 
The reason for these two picks is mainly due to cost and ease of maintenance. 
Each PPK event is eventually written to a google sheet dedicated to a specific month for all VOID, and a separate sheet for the pilot's corp. This allows corp's to build thier own unique systems without having to re-create all the other parts.

## Adding PPK Events via Discord

For steps 3 in [pilot workflow](Pilot-Workflow) the Discord bot listens for KMs, creates a row for the KM, then creates PPK buttons. Note that PPK is assigned to an in-game clone, not a Discord user. 


``` C#
// Wire up events to process messages and button clicks
_client.MessageCommandExecuted += handlers.ClientOnMessageCommandExecuted;
_client.ButtonExecuted += handlers.ClientOnButtonExecuted;

   internal Task ClientOnMessageCommandExecuted(SocketMessageCommand messageCommand)
    {
        // Code omitted to check certain things about the buttons. The killMailId is parsed from the message.
        
        this._logger.Log($"Creating buttons.");
        ComponentBuilder builder = this.CreatePpkButton(killMailId);
        messageCommand.RespondAsync($"Add a PPK Event to Killmail ID [{killMailId}] [<{messageCommand.Data.Message.GetJumpUrl()}>]:", components: builder.Build());
        return Task.CompletedTask;
    }

 internal ComponentBuilder CreatePpkButton(string killMailId)
    {
       // Note the use of the custom Id to plumb through the killmail Id.
        return new ComponentBuilder()
            .WithButton("PARTICIPANT", $"{KillMailParticipantType.PARTICIPANT}-{killMailId}", ButtonStyle.Primary)
            .WithButton("TACKLE", $"{KillMailParticipantType.TACKLE}-{killMailId}", ButtonStyle.Danger)
            .WithButton("DICTOR", $"{KillMailParticipantType.DICTOR}-{killMailId}", ButtonStyle.Danger)
            .WithButton("LOGI", $"{KillMailParticipantType.LOGI}-{killMailId}", ButtonStyle.Success)
            .WithButton("SCOUT", $"{KillMailParticipantType.SCOUT}-{killMailId}", ButtonStyle.Danger)
            .WithButton("FC", $"{KillMailParticipantType.FC}-{killMailId}", ButtonStyle.Danger);

    }
    
    internal Task ClientOnButtonExecuted(SocketMessageComponent messageComponent)
    {
        // Get the KM Id, and the PPK Event selected by parsing the custom Id
        string dataCustomId = messageComponent.Data.CustomId;
        this._logger.Log($"{messageComponent.User.Mention} has clicked the button with id [{dataCustomId}]");
        DiscordId pilotId = (DiscordId)messageComponent.User.Id;
        string[] result = dataCustomId.Split("-");
        string type = result[0];
        string killMailId = result[1];

        this._logger.Log($"[{type}]-[{killMailId}] button pressed.");
        
        // Look up the clones to assign PPK to
        IEnumerable<Clone> clones = this._cloneRepository.GetClonesByDiscordId(pilotId);
        IEnumerable<Clone> enumerable = clones.ToList();

        if (enumerable.Count() == 1)
        {
            Clone clone = enumerable.Single();
            this._logger.Log($"Pilot with single clone {clone.cloneName} being added to PPK");
            
            // Store the PPK event
            return this.AddPpkForClone(messageComponent, clone, type, killMailId, pilotId);
        }
     
     // Some more code ommited for brevity to deal with when the user has more than one registered clone.    
    }
```


## Further Discord Commands

Because things can go wrong, you also need commands for the following locked to certian discord user considered admins:
* Re-parse and re-create the buttons for a KM message.
* Manually override any values parsed wrong.
* Manually remove or add a PPK event on behalf of another pilot.


So that's a highlevel overview of VOID's PPK Events, with some details about selecting PPK Events via Discord.

<a href="https://www.buymeacoffee.com/sarpedontdw" target="_blank"><img src="https://cdn.buymeacoffee.com/buttons/default-orange.png" alt="Buy Me A Coffee" height="41" width="174"></a>

