// ==UserScript==
// @name         Cookie Clicker Auto Player
// @namespace    http://tampermonkey.net/
// @version      2.016.1
// @description  Auto-plays the game with close to human performance. Controllable via bakery name.
// @author       Orc JMR
// @grant        none
// ==/UserScript==

var Bot = {};
Bot._versionCode = "2.016.1"; // remember to update this to the script version above


// ---- Check and modify the following numbers, if they don't suit your stage / playstyle

Bot._clickBigCookieWhenCpsBuffIs=7;      // 40 means 4000% CpS, for example a building special from 390 buildings.

Bot._clicksPerSecond=100;                 // this bot is designed to work at low clicks/s, like 7.
                                        // Setting it to 200 can have unfun consequences, I've never tried, warranty void.


// ---- End of numbers. Here goes bot's code, don't go there unless you do want to break/fix things.

Bot.lastName="";
Bot.origVersion="";

Bot.initHandle=null;
Bot.initLoop=function()
{
    Bot.origVersion=l("versionNumber").textContent;
    if(Bot.origVersion)
    {
        Bot.origVersion += " <span style=\"font-size:14px\"> Bot v." + Bot._versionCode + "</span>"
        window.clearInterval(Bot.initHandle);
        Bot.initHandle = null;
        Bot.recalcHandle = window.setInterval(Bot.recalcLoop, 500);
        return;
    }
}

Bot.gardenRecalcTime=0;
Bot.recalcHandle=null;
Bot.recalcLoop=function()
{
    if(Bot.lastName != Game.bakeryName)
    {
        // Turn the bot on/off and change actions
        var name = Game.bakeryName;
        if(!name.endsWith("Bot"))
        {
            if(Bot.clickHandle)
            {
                window.clearInterval(Bot.clickHandle);
                Bot.clickHandle = null;
            }
            l("versionNumber").innerHTML=Bot.origVersion;
            Bot.shouldBuySeason = false;
            Bot.shouldPickLumps = false;
            Bot.shouldKeepGarden = false;
            Bot.shouldClickCookie = false;
        }
        else
        {
            var startMode = (name.indexOf("Launch")>=0);
            var runMode = (name.indexOf("Cruise")>=0);
            Bot.doesClickShimmers = startMode || runMode || (name.indexOf("GC")>=0);
            Bot.doesClickUpgrades = startMode || (name.indexOf("Buy")>=0);
            Bot.doesClickBuildings = Bot.doesClickUpgrades && !!Bot.objectLimits; // disabled if bought everything
            Bot.doesClickSpells = runMode || (name.indexOf("Magic")>=0);
            Bot.shouldClickCookie = startMode || runMode || (name.indexOf("Click")>=0);
            Bot.shouldBuySeason = runMode || (name.indexOf("Fool")>=0);
            Bot.shouldPickLumps = startMode || runMode || (name.indexOf("Lump")>=0);
            Bot.shouldKeepGarden = name.indexOf("Crop")>=0;

            var botModeString=(
                (Bot.shouldClickCookie?"Clicks/":"")+
                (Bot.doesClickShimmers?"GCs/":"")+
                (Bot.doesClickUpgrades?"Buys/":"")+
                (Bot.shouldBuySeason?"Fools/":"")+
                (Bot.doesClickSpells?"Casts/":"")+
                (Bot.shouldPickLumps?"Lumps/":"")+
                (Bot.shouldKeepGarden?"Toils/":"")).slice(0,-1);
            l("versionNumber").innerHTML=Bot.origVersion+"<br/><span style=\"font-size:14px\">Auto-"+botModeString+"</span>";

            if(!Bot.clickHandle)
                Bot.clickHandle = window.setInterval(Bot.clickLoop, 1000/Bot._clicksPerSecond);
        }
        Bot.lastName = name;
    }

    // recalculate stuff not needed each click
    var t = Date.now();
    Bot.doesClickSeason = Bot.shouldBuySeason && Bot.canBuySeason(Bot._seasonToBuy);
    Bot.doesClickLumps = Bot.shouldPickLumps && Game.canLumps() && t>=Game.lumpT+Game.lumpRipeAge;
    Bot.doesClickGarden &= Bot.shouldKeepGarden; // deactivate garden tasks, or they will still get executed
    if(Bot.shouldClickCookie || Bot.shouldKeepGarden)
    {
        Bot.isClickBuffed = false;
        Bot.cpsMultiplier = 1;
        for (var i in Game.buffs)
        {
            var me = Game.buffs[i];
            if (me.name == 'Cursed finger') Bot.isClickBuffed = true;
            if (typeof me.multClick != 'undefined' && me.multClick > 1) Bot.isClickBuffed = true;
            if (typeof me.multCpS != 'undefined') Bot.cpsMultiplier *= me.multCpS;
        }
    }
    if(Bot.shouldKeepGarden && t>Bot.gardenRecalcTime)
    {
        var M=Game.Objects['Farm'].minigame;
        if(M)
        {
            Bot.gardenRecalcTime = M.nextStep + 1000; // 1 sec after tick
            Bot.gardenHarvestAtTopCps = [];
            Bot.gardenHarvestAtHiCps = [];
            Bot.gardenHarvestNow = [];
            Bot.gardenPlant = [];

            // == Juicy Queenbeet strategy ==
            var queenbeet = M.plants['queenbeet'];
            var juicybeet = M.plants['queenbeetLump'];
            var bakeberry = M.plants['bakeberry'];
            var zones = [{},{},{},{}];
            var zoneCoords = [[1,1],[1,3],[3,1],[3,3]];

            // -- Detection
            // each target plot
            for(var z=0;z<4;z++)
            {
                if(M.getTile(zoneCoords[z][0],zoneCoords[z][1])[0]-1==juicybeet.id)
                    zones[z].gotcha=true;
                Bot.processGardenTile(M,zoneCoords[z][0],zoneCoords[z][1],[juicybeet.id],[2]);
            }
            // surrounding plots
            for(var y=0;y<5;y++)
                for(var x=0;x<5;x++)
                {
                    if((x==1 || x==3) && (y==1 || y==3)) // skip target plots
                        continue;

                    var tile = M.getTile(x,y);
                    if(tile[0]!=0)
                    {
                        if(x<3 && y<3) zones[0].nonEmpty=true;
                        if(x<3 && y>1) zones[1].nonEmpty=true;
                        if(x>1 && y<3) zones[2].nonEmpty=true;
                        if(x>1 && y>1) zones[3].nonEmpty=true;
                    }
                    if(tile[0]-1!=queenbeet.id)
                    {
                        if(x<3 && y<3) zones[0].expired=true;
                        if(x<3 && y>1) zones[1].expired=true;
                        if(x>1 && y<3) zones[2].expired=true;
                        if(x>1 && y>1) zones[3].expired=true;
                    }
                }
            var gotchaCount=0;
            var expiredCount=0;
            var emptyCount=0;
            for(var z=0;z<4;z++)
            {
                if(zones[z].gotcha)
                    gotchaCount++;
                else
                {
                    if(zones[z].expired) expiredCount++;
                    if(!zones[z].nonEmpty) emptyCount++;
                }
            }
            var allExpired = gotchaCount+expiredCount==4;
            var allEmpty = gotchaCount+emptyCount==4;

            // -- Spare plots for Bakeberries first
            for(var y=0;y<6;y++)
                Bot.processGardenTile(M,5,y,[bakeberry.id],[0],bakeberry.id);
            for(var x=0;x<5;x++)
                Bot.processGardenTile(M,x,5,[bakeberry.id],[0],bakeberry.id);

            // if got lump - gotcha, harvest @ top, plant berries
            // if not lump - kill
            // if full circle - waiting/active, noop
            // if not full circle - expired, harvest @ top, wait
            // if all non-gotcha plots are expired - harvest @ high
            // if all non-gotcha plots are ready - plant beets

            // -- Action
            Bot.processJuicyStrategy(M,zones, // dead center
                allEmpty,allExpired,[[2,2]]);
            Bot.processJuicyStrategy(M,[zones[0],zones[1]], // middle left
                allEmpty,allExpired,[[0,2],[1,2]]);
            Bot.processJuicyStrategy(M,[zones[0]], // top left
                allEmpty,allExpired,[[0,0],[0,1],[1,0]]);
            Bot.processJuicyStrategy(M,[zones[0],zones[2]], // top middle
                allEmpty,allExpired,[[2,0],[2,1]]);
            Bot.processJuicyStrategy(M,[zones[2]], // top right
                allEmpty,allExpired,[[3,0],[4,0],[4,1]]);
            Bot.processJuicyStrategy(M,[zones[2],zones[3]], // middle right
                allEmpty,allExpired,[[3,2],[4,2]]);
            Bot.processJuicyStrategy(M,[zones[3]], // bottom right
                allEmpty,allExpired,[[3,4],[4,3],[4,4]]);
            Bot.processJuicyStrategy(M,[zones[1],zones[3]], // bottom middle
                allEmpty,allExpired,[[2,3],[2,4]]);
            Bot.processJuicyStrategy(M,[zones[1]], // bottom left
                allEmpty,allExpired,[[0,3],[0,4],[1,4]]);
            // == end of Juicy Queenbeet strategy ===== //

            /*/ == Full Bakeberry strategy ==
            var bakeberry = M.plants['bakeberry'];
            for(var y=0;y<6;y++)
                for(var x=0;x<6;x++)
                {
                    Bot.processGardenTile(M,x,y,[bakeberry.id],[0],bakeberry.id);
                }
            // ======= /*/

            Bot.doesClickGarden = (Bot.gardenHarvestAtHiCps.length +
                                    Bot.gardenHarvestAtTopCps.length +
                                    Bot.gardenHarvestNow.length +
                                    Bot.gardenPlant.length) > 0;
        }
    }
}

Bot.clickCooldown=0;
Bot.clickHandle=null;
Bot.clickedGarden=false;
Bot.clickLoop=function()
{
    if(Bot.clickCooldown > 0)
    {
        Bot.clickCooldown--;
        return;
    }

    if(Bot.shouldClickCookie && (Bot.isClickBuffed || Bot.cpsMultiplier >= Bot._clickBigCookieWhenCpsBuffIs))
        Bot.doClickCookie();
}

Bot.doClickCookie=function()
{
    var mx = Game.mouseX;
    var my = Game.mouseY;
    Game.mouseX=Game.cookieOriginX;
    Game.mouseY=Game.cookieOriginY;
    Game.ClickCookie();
    Game.mouseX = mx;
    Game.mouseY = my;
}


Bot.initHandle = window.setInterval(Bot.initLoop, 1000);
