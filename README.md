
# WhatsApp Bot Interaction with Home Assistant Assistant and User/Group ID Storage Automation

This guide explains how to set up an interactive WhatsApp bot that communicates with Home Assistant's Assistant and automatically captures and stores WhatsApp user and group IDs from incoming WhatsApp messages. This setup allows you to both automate conversations through Home Assistant's Assistant and maintain a dynamic record of users and groups.

## Overview

This blueprint enables your WhatsApp bot to interact directly with Home Assistant's Assistant, allowing for real-time conversation handling. In addition, it automatically tracks users and group IDs, updating your Home Assistant configuration with each unique ID that interacts with your bot.

## Prerequisites

To use this automation, ensure you have the following:
- The [WhatsApp integration add-on by Giuseppe Castaldo](https://github.com/giuseppecastaldo/ha-addons/tree/main/whatsapp_addon) configured in Home Assistant. This add-on is required for receiving WhatsApp messages in Home Assistant.
- The automation blueprint and the appropriate permissions set up to trigger on incoming WhatsApp messages.

## Quick Import

You can quickly import this blueprint into your Home Assistant instance using the button below:

[![Open your Home Assistant instance and show the blueprint import dialog with this blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https://raw.githubusercontent.com/phoenix-blue/HA_Blueprints/refs/heads/main/blueprints/automation/phoenix-blue_whatsapp-bot/whatsapp-bot.yaml)

## Entities

We use two `input_select` entities to store the WhatsApp user and group IDs. These dropdown lists will automatically update with new IDs as messages are received from different users or groups.

### 1. `input_select.whatsapp_user_ids`

Stores the WhatsApp user IDs of individuals who send messages to your bot. This entity will automatically update when a new user sends a message.

Define this `input_select` in your configuration:

```yaml
input_select:
  whatsapp_user_ids:
    name: WhatsApp User IDs
    options:
      - None
    icon: mdi:account
```

### 2. `input_select.whatsapp_chat_ids`

Stores the WhatsApp group IDs of groups where your bot receives messages. This entity will automatically update when a new group ID is detected.

Define this `input_select` in your configuration:

```yaml
input_select:
  whatsapp_chat_ids:
    name: WhatsApp Group IDs
    options:
      - None
    icon: mdi:account-group
```

## Automation

This automation listens for incoming WhatsApp messages, checks whether the sender is a user or a group, and engages with Home Assistant's Assistant. Depending on the ID type, it updates the appropriate `input_select` with the new ID if it hasn’t been recorded before.

### Automation Code

Add the following automation code to your Home Assistant configuration to enable automatic storage of new WhatsApp user and group IDs and interact with Home Assistant’s Assistant:

```yaml
alias: "Automation to Save New WhatsApp User and Group IDs"
description: "Automation to automatically store new WhatsApp user and group IDs and interact with the Home Assistant Assistant"
trigger:
  - platform: event
    event_type: new_whatsapp_message
condition: []
action:
  - choose:
      - conditions:
          - condition: template
            value_template: >
              {{ trigger.event.data["key"]["remoteJid"].endswith("@s.whatsapp.net") }}
        sequence:
          - service: input_select.set_options
            data:
              entity_id: input_select.whatsapp_user_ids
              options: >
                {% set new_id = trigger.event.data["key"]["remoteJid"].split("@")[0] %}
                {% set current_ids = state_attr('input_select.whatsapp_user_ids', 'options') %}
                {% if new_id not in current_ids %}
                  {{ (current_ids + [new_id]) | list }}
                {% else %}
                  {{ current_ids }}
                {% endif %}
          - service: conversation.process
            data:
              text: "{{ trigger.event.data['message']['conversation'] | default('') }}"
              conversation_id: "whatsapp_{{ trigger.event.data['key']['remoteJid'] }}"
              agent_id: conversation.home_assistant
            response_variable: response
          - service: whatsapp.send_message
            data:
              clientId: whatsapp
              to: "{{ trigger.event.data['key']['remoteJid'] }}"
              body:
                text: "{{ response.response.speech.plain.speech }}"
      - conditions:
          - condition: template
            value_template: >
              {{ trigger.event.data["key"]["remoteJid"].endswith("@g.us") }}
        sequence:
          - service: input_select.set_options
            data:
              entity_id: input_select.whatsapp_chat_ids
              options: >
                {% set new_id = trigger.event.data["key"]["remoteJid"].split("@")[0] %}
                {% set current_ids = state_attr('input_select.whatsapp_chat_ids', 'options') %}
                {% if new_id not in current_ids %}
                  {{ (current_ids + [new_id]) | list }}
                {% else %}
                  {{ current_ids }}
                {% endif %}
mode: single
```

### Explanation of the Automation

- **Trigger**: The automation listens for the `new_whatsapp_message` event, triggered each time a WhatsApp message is received.
- **Conditions**: The `choose` block evaluates the message source to determine if it’s a user or group.
  - **User ID**: If the `remoteJid` ends with `@s.whatsapp.net`, it identifies the message as coming from an individual user. The user’s ID (phone number) is then extracted and stored in `input_select.whatsapp_user_ids` if it’s not already there.
  - **Group ID**: If the `remoteJid` ends with `@g.us`, it identifies the message as coming from a group. The group ID is then extracted and stored in `input_select.whatsapp_chat_ids` if it’s not already there.
- **Actions**: For each new user or group ID, the automation updates the appropriate `input_select` entity with the new ID, ensuring there are no duplicate entries. For users, it also processes their message through Home Assistant's Assistant, and the response is sent back via WhatsApp.

## How It Works

When a WhatsApp message is received, this automation will:
1. Check if the message is from a user or a group.
2. If it’s from a user, it:
   - Extracts the user’s ID (without the `@s.whatsapp.net` suffix) and adds it to the `whatsapp_user_ids` list if it’s new.
   - Processes the message through Home Assistant's Assistant and sends the response back to the user.
3. If it’s from a group, it extracts the group ID (without the `@g.us` suffix) and adds it to the `whatsapp_chat_ids` list if it’s new.

This setup allows you to maintain a real-time list of WhatsApp users and groups interacting with your bot and provides interactive responses through Home Assistant’s Assistant.

## Customization

- **Initial Options**: You can change the default option (currently set to `None`) in the `input_select` entities to suit your needs.
- **Icons**: Customize the icons for `whatsapp_user_ids` and `whatsapp_chat_ids` using any icon from [Material Design Icons](https://materialdesignicons.com/).

## Troubleshooting

- Ensure that the WhatsApp integration is working correctly and messages are being received by Home Assistant.
- Verify that `input_select.whatsapp_user_ids` and `input_select.whatsapp_chat_ids` are correctly defined in your configuration.
- Check the automation logs in Home Assistant if the IDs are not updating as expected.

## Credits

This automation is based on the original concept of the [Telegram bot blueprint by marc-romu](https://github.com/marc-romu/home-assistant_blueprints/tree/main/blueprints/automation/marc-romu_telegram-bot).

---

By following this guide, you will have an automation that dynamically tracks new WhatsApp user and group IDs, interacts with Home Assistant’s Assistant in real-time, and keeps your `input_select` entities updated. This can be especially useful for creating custom notifications, filtering messages, and logging interactions.
