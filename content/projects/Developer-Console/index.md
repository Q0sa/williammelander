+++
date = '2026-03-19'
draft = false
title = 'Developer Console'

summary = """Developer Console Tool used for debugging, testing, and active development."""

tags = ["C++", "Tool", "Solo", "Dear ImGUI"]

+++
![DevConsShowcase](/img/DeveloperConsole.webp)

<h2>
Here is a Source Engine style Developer Console! 
</h2>

---

<h2>The Problem: <i> Debug and Feature Testing Clutter </i></h2>

<h5>

Each game project it was common practice to throw in an if statement here and there for the specific purpose of testing or debugging.
<br>

This required us to:
- Actively communicate what each debug keybind was and what they did.
- Needing to hunt down deprecated (or overlapping) debug binds. 
- No support for parameter handling without doing a bunch of additional work.

This bugged me quite a bit, as well as made it harder for everyone to properly test our games!
</h5>

<h1></h1>
<h2>The Solution: <i> The Classic Developer Console! </i></h2> 
<h5>

Since its inclusion the Developer Console has been used by everyone in my group! From the camera to rendering settings, here are some example commands:

- `gfx_volumetric_fog`
  - Toggles volumetric fog (on/off)
- `phys_render_characters_radius`
  - Physics Engine, Render Characters currently whithin the radius [radius, state] <br> 
    eg: phys_render_characters_radius 500.f true
- `player_add_spell`
  - Unlock a spell, leave empty for all spells [QuickSpell, BurstSpell, GrenadeSpell, HeavySpell]
- `noclip`
  - Toggles noclip (freefly mode for player)

</h5>


---

### Overall Structure

The Developer Console isn't a class but rather a namespace that can be globally accessed. My motivation of doing this is to keep the console flexible and allow commands to be easily registered, also allowing it to operate outside of the game engine.

With slight modification it can be turned into a class that is owned by the game engine as a `std::unique_ptr`, but this currently isn't the case.

---

### Command Registration

```cpp
void RegisterConsoleCommand(const std::string& aCommandName, 
   const std::function<bool(const std::vector<std::string>&)>& aCommandFunction,
   const std::string& aHelpText)
{
    if (!locRegisteredCommands.try_emplace(aCommandName, 
                                           ConsoleCommand
                                           {
                                             .command = aCommandFunction, 
                                             .helpText = aHelpText
                                           }).second)
    {
        // This is specifically to handle scene load of unique pointers that get created 
        // mainly due to the [this] lambda registering the pre unique pointer object, 
        // not the actual object used so this exists purely to ensure that the [this] 
        // pointer points to the right object
        locRegisteredCommands[aCommandName].command = aCommandFunction; 
    }
}
```

Above is the function used to register a command. If the `try_emplace` should fail there is only one reason which due to there already being a command of the same name. However if this occurs the already existing command get the latest version of the function memory address. 

That being said command registration should only exist in objects that are guaranteed to not move memory position, but that is only if `[this]` is captured. An example of playing this dangerous game is with the debug commands for the Player from the group project Walter Volt (page does not exist yet). 

<details style="border: 0px solid #fd9800; border-radius: 8px; padding: 12px; background-color:rgba(124, 45, 18, 0.45);">
<summary><b>Click here to view : Command Registration Example</b></summary>

```cpp
DeveloperConsole::RegisterConsoleCommand("noclip", [this](const std::vector<std::string>& aArguments)
{
    auto* physics = Engine::GetGameplayEngine().GetSystem<PhysicsSystem>();
    if (aArguments.empty()) //handle empty argument list
    {
        myNoClipActive = !myNoClipActive; // here is danger, references class member
                                          // player doesn't move in memory so this is fine, until its not 

        physics->SetPlayerNoClip(myNoClipActive);
        
        DeveloperConsole::OutputToDevConsole("[[noclip toggled]]");
        return true;
    }
        
    bool aState;
    if (DeveloperConsole::TryParse<bool>(aArguments.at(0), aState)) //parse argument
    {
        myNoClipActive = aState; //here is danger once again
        physics->SetPlayerNoClip(aState);
        
        DeveloperConsole::OutputToDevConsole("[[noclip set to " + aArguments.at(0) + "]]");
        return true;
    }
        
    //If both don't succeed return failure
    return false;
}, "toggles noclip (freefly mode for player)");
```
</details>
<br>
<details style="border: 0px solid #fd9800; border-radius: 8px; padding: 12px; background-color:rgba(124, 45, 18, 0.45);">
<summary><b>Click here to view : TryPase for Argument Parsing</b></summary>

```cpp
template<typename T>
    bool TryParse(const std::string& aString, T& aOutVar)
    {
        //Special handling for true false inputs
        if constexpr (std::is_same_v<T, bool>)
        {
            if (aString == "true" || aString == "1")
            {
                aOutVar = true;
                return true;
            }
            
            if (aString == "false" || aString == "0")
            {
                aOutVar = false;
                return true;
            }
            
            return false;
        }
        
        //Avoid unnecessary copies
        else if constexpr (std::is_same_v<T, std::string>)
        {
            aOutVar = aString;
            return true;
        }
        else
        {
            std::istringstream stream(aString);
            return !(stream >> aOutVar).fail();
        }
    }
```

</details>

--- 

### Command Storage
```cpp
struct ConsoleCommand 
{
    std::function<bool(const std::vector<std::string>&)> command;
    std::string helpText;
};

std::map<std::string, ConsoleCommand> locRegisteredCommands;
```

`locRegisteredCommands` : Keeps track of all commands, sorted by the command name. <br>
`ConsoleCommand` : The struct that is the command itself as well as the help text (which is a description of what the command does).

I found this was the most straight forward way of storing all the commands, an `std::map` that keeps the commands sorted (useful for displaying suggestions and keeping commands grouped).

`std::function` does have some overhead but is fine in this application due to it being used for a console command, which shouldn't be called ever frame. 

---

### From Console to In-Game Flowchart
Here is the step by step overview of how commands are processed and applied.

<pre style="text-align: center;">
      ┌──────────────────────────┐          
      │Input Command + Parameters│          
      └─────────────┬────────────┘          
                    │                       
  ┌─────────────────▼───────────────────┐   
  │Convert input string to a std::vector│   
  │of strings, using ' ' as a delimiter │   
  └─────────────────┬───────────────────┘   
                    │                       
   ┌────────────────▼───────────────────┐   
   │Extract command name from the vector│   
   │then removes it from the vector,    │   
   │vector is now the parameter list.   │   
   └────────────────┬───────────────────┘   
                    │                       
   ┌────────────────▼──────────────────┐    
   │Pass the extracted command name and│    
   │parameter list of strings into the │    
   │TriggerCommand function.           │    
   └────────────────┬──────────────────┘    
                    ▼                       
       [[Now within TriggerCommand]]        
                    │                       
    ┌───────────────▼───────────────────┐   
    │Check if the command exists in the │   
    │locRegisteredCommands std::map     │   
    └─┬───────────────────┬─────────────┘   
      │                   │Failure to       
      │                   │locate command   
      │   ┌───────────────▼───────────┐     
      │   │ Output failure to console │     
      │   │ "Command does not exist!" │     
      │   └───────────────────────────┘     
      │                                     
      └─────────────┐                       
                    │ Found Command!        
      ┌─────────────▼─────────────────┐     
      │Pass to the registered function│     
      │which then executes and handles│     
      │parameter list                 │     
      └──┬──────────────────────┬─────┘     
         │Success!              │Failure    
┌────────▼──────────┐ ┌─────────▼──────────┐
│Function outputs   │ │ Output failure     │
│success to console │ │ suggesting to use  │
└───────────────────┘ │ the 'help' command │
                      └────────────────────┘ 
</pre>

---

<h3> 
That's the Developer Console, it still needs some additional handling and such but so far it's worked like a charm!
</h3>