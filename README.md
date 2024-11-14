
# WhatsApp User and Group ID Storage Automation

This guide explains how to automatically capture and store new WhatsApp user and group IDs from incoming WhatsApp messages in Home Assistant. By following this guide, you will set up `input_select` entities for users and groups, and an automation that updates these lists whenever new IDs are detected.

## Prerequisites

To use this automation, ensure you have the following:
- The [WhatsApp integration add-on by Giuseppe Castaldo](https://github.com/giuseppecastaldo/ha-addons/tree/main/whatsapp_addon) configured in Home Assistant. This add-on is required for receiving WhatsApp messages in Home Assistant.
- The automation blueprint and the appropriate permissions set up to trigger on incoming WhatsApp messages.

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

This automation listens for incoming WhatsApp messages and checks whether the sender is a user or a group. Depending on the ID type, it updates the appropriate `input_select` with the new ID if it hasn’t been recorded before.

### Automation Code

Add the following automation code to your Home Assistant configuration to enable automatic storage of new WhatsApp user and group IDs:

```yaml
alias: "Automation to Save New WhatsApp User and Group IDs"
description: "Automation to automatically store new WhatsApp user and group IDs"
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
- **Actions**: For each new user or group ID, the automation updates the appropriate `input_select` entity with the new ID, ensuring there are no duplicate entries.

## How It Works

When a WhatsApp message is received, this automation will:
1. Check if the message is from a user or a group.
2. If it’s from a user, it extracts the user’s ID (without the `@s.whatsapp.net` suffix) and adds it to the `whatsapp_user_ids` list if it’s new.
3. If it’s from a group, it extracts the group ID (without the `@g.us` suffix) and adds it to the `whatsapp_chat_ids` list if it’s new.

This setup allows you to maintain a real-time list of WhatsApp users and groups interacting with your bot.

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

By following this guide, you will have an automation that dynamically tracks new WhatsApp user and group IDs, keeping your `input_select` entities updated in real-time. This can be especially useful for creating custom notifications, filtering messages, or logging interactions.
