![niflveil-logo-complete](https://github.com/user-attachments/assets/74032954-8770-460d-87f0-c2057328197a)# NiflVeil
Hiding your dirty little secrets since 2024

From the creator of [StygianSift](https://github.com/Mauitron/StygianSift)


# NiflVeil
A minimalistic window minimizer for Hyprland, named after Niflheim's mystical veil of mists where things vanish from sight but remain within reach.
If you enjoy NiflVeil or want to see more tools from my workshop in Niflheim, consider buying me some [mead](https://buymeacoffee.com/charon0)  🍺

## Lost in the Mists? Let this paragon of sanity help you out!

Greetings, mortals! I am the keeper of the mists, formerly known as that boat guy from the warmer realms. After a... slight navigational error involving the world tree and some, exceptionally terrible, advice from an, exceptionally rude, squirl, I've found myself in these frigid Norse lands. But fear not! I've turned my talent for ferrying lost souls into something more useful - helping you manage your lost windows!

## 🌫️ Features

![output_optimized](https://github.com/user-attachments/assets/0bf280bf-a41c-460a-8c34-a0b86a5d2dc8)

### Core Features
- **Window Veiling**: Send your windows to Niflheim's mists, retrieve them when needed
- **Wofi Integration**: To let you peer across the veil
- **Waybar Support**: Keep track of your hidden treasures
- **Window State Persistence**: Like Yggdrasil's roots, always remembering

### Minimizing Magic
```bash
Super + M to send window to the mists
Super + I to part the veil and see what lurks beyond
```

## 🛠️ Dependencies (A Smith's Requirements)

- hyprland (your vessel through the desktop seas)
- wofi (because even Vikings need menus)
- jq (for parsing the runes... I mean, JSON)
- waybar (optional, but recommended for keeping track of your veiled windows)

## Installation


## The Script

1. Copy the following scirpt:

```bash
#!/bin/bash

# Core state management
CACHE_DIR="/tmp/minimize-state"
CACHE_FILE="$CACHE_DIR/minimized_windows.json"

# Ensure cache directory exists
mkdir -p "$CACHE_DIR"

# Initialize cache file if it doesn't exist
if [ ! -f "$CACHE_FILE" ] || ! jq empty "$CACHE_FILE" 2>/dev/null; then
    echo '[]' > "$CACHE_FILE"
fi

minimize() {
    window_info=$(hyprctl activewindow -j)
    if [ $? -eq 0 ] && [ -n "$window_info" ] && [ "$(echo "$window_info" | jq -r '.class')" != "null" ]; then
        class_name=$(echo "$window_info" | jq -r '.class')
        
        # Don't minimize wofi
        if [ "$class_name" = "wofi" ]; then
            exit 1
        fi
        
        window_title=$(echo "$window_info" | jq -r '.title')
        window_addr=$(echo "$window_info" | jq -r '.address')
        short_addr=$(echo "$window_addr" | tail -c 5)
        
        # Create window info object
        window_info=$(echo "$window_info" | jq --arg title "$class_name - $window_title [$short_addr]" \
            '{address: .address, display_title: $title, class: .class, original_title: .title}')
        
        # Move window to special workspace
        if hyprctl dispatch movetoworkspacesilent special:minimum,address:$window_addr; then
            # Update cache file
            if [ -s "$CACHE_FILE" ]; then
                echo "$window_info" | jq -s ". + $(cat "$CACHE_FILE")" > "$CACHE_FILE.tmp"
            else
                echo "$window_info" | jq -s > "$CACHE_FILE.tmp"
            fi
            mv "$CACHE_FILE.tmp" "$CACHE_FILE"
            
            # Signal waybar if you're using it
            pkill -RTMIN+8 waybar 2>/dev/null || true
        fi
    fi
}

restore() {
    window_id="$1"
    if [ -n "$window_id" ]; then
        current_ws=$(hyprctl activeworkspace -j | jq -r '.id')
        
        if hyprctl clients -j | jq -e ".[] | select(.address == \"$window_id\")" >/dev/null; then
            # Move window to current workspace and focus it
            if hyprctl dispatch movetoworkspace "$current_ws,address:$window_id"; then
                hyprctl dispatch focuswindow "address:$window_id"
                
                # Update cache file
                jq "map(select(.address != \"$window_id\"))" "$CACHE_FILE" > "$CACHE_FILE.tmp"
                mv "$CACHE_FILE.tmp" "$CACHE_FILE"
                
                # Signal waybar if you're using it
                pkill -RTMIN+8 waybar 2>/dev/null || true
            fi
        else
            # Clean up if window no longer exists
            jq "map(select(.address != \"$window_id\"))" "$CACHE_FILE" > "$CACHE_FILE.tmp"
            mv "$CACHE_FILE.tmp" "$CACHE_FILE"
            pkill -RTMIN+8 waybar 2>/dev/null || true
        fi
    fi
}

show_menu() {
    if [ ! -s "$CACHE_FILE" ]; then
        echo "No minimized windows"
        exit 0
    fi

    # Create menu content
    menu_content=$(jq -r '.[].display_title' "$CACHE_FILE")
    
    # Show wofi menu
    selected=$(echo -e "$menu_content" | wofi \
        --show dmenu \
        --prompt "Minimized Windows" \
        --width 800 \
        --height 400 \
        --cache-file /dev/null \
        --style "$HOME/.config/wofi/style.css" \
        --hide-scroll \
        --define search_triggered=false \
        --define early_exit=true \
        --define immediate_activation=true \
        --define single_click=true)
    
    if [ -n "$selected" ]; then
        window_id=$(jq -r --arg title "$selected" '.[] | select(.display_title == $title) | .address' "$CACHE_FILE")
        if [ -n "$window_id" ]; then
            restore "$window_id"
        fi
    fi
}

case "$1" in
    "minimize")
        minimize
        ;;
    "restore")
        if [ -n "$2" ]; then
            restore "$2"
        else
            show_menu
        fi
        ;;
    "show")
        if [ -s "$CACHE_FILE" ]; then
            count=$(jq length "$CACHE_FILE")
            echo "{\"text\":\"󰖰 $count\",\"class\":\"has-windows\",\"tooltip\":\"$count minimized windows\"}"
        else
            echo "{\"text\":\"󰖰\",\"class\":\"empty\",\"tooltip\":\"No minimized windows\"}"
        fi
        ;;
    *)
        echo "Usage: $0 {minimize|restore [window_id]|show}"
        exit 1
        ;;
esac
```

2. Create the wofi style file:
```bash
mkdir -p ~/.config/wofi
cat > ~/.config/wofi/style.css << 'EOL'
window {
    background-color: #2e3440;
    border: 2px solid #4c566a;
    border-radius: 8px;
}

#input {
    border: none;
    background: #3b4252;
    border-radius: 4px;
    margin: 8px;
    padding: 8px;
    color: #eceff4;
    font-family: "JetBrainsMono Nerd Font";
}

#outer-box {
    margin: 0;
    border: none;
}

#inner-box {
    margin: 4px;
    border: none;
}

#entry {
    padding: 8px;
    margin: 4px;
    border-radius: 4px;
    background: #3b4252;
}

#entry:selected {
    background: #4c566a;
    border: 1px solid #88c0d0;
}

#text {
    color: #eceff4;
    font-family: "JetBrainsMono Nerd Font";
}
EOL
```

3. Add these bindings to your Hyprland config (~/.config/hypr/hyprland.conf):
```conf
# NiflVeil bindings
bind = $mainMod, M, exec, niflveil minimize
bind = $mainMod, I, exec, niflveil restore
```

4. (Optional) Add Waybar integration by adding this to your Waybar config:
```jsonc
{
    "custom/niflveil": {
        "format": "{}",
        "exec": "niflveil show",
        "on-click": "niflveil restore",
        "return-type": "json",
        "interval": "once",
        "signal": 8
    }
}
```

And add this to your Waybar style.css:
```css
#custom-niflveil {
    padding: 0 10px;
    color: #88c0d0;
}

#custom-niflveil.empty {
    color: #4c566a;
}
```

## Usage

- Press `Super + M` to minimize the current window
- Press `Super + I` to show the restore menu
- Click the Waybar module (if configured) to show minimized windows


## Why Trust Your Windows to a Lost Ferryman?

Look, I might have taken a wrong turn at the River Styx and ended up in Niflheim, but I know a thing or two about managing lost things. Whether they're souls or windows, the principle is the same - keep track of them and make sure they can find their way back home!

## 🍺 Support the Development

If you find this tool useful, consider buying me some mead! I could use it in these cold Norse realms. Also, the heating bill in Niflheim is astronomical.

## 🌟 Acknowledgments

- The Hyprland community for not pointing out that I'm clearly lost
- Nordic theme creators for making me feel more at home in these cold lands
- That absolute tit Ratatoskr who gave me directions! (I should have known better)


P.S. No refunds, exchanges, or soul-backsies. The exchange rate between Obols and Norse currency is terrible anyway.

P.P.S. If you see a confused-looking boat anywhere in the world tree, please let me know. I'm starting to... yearn for it.
