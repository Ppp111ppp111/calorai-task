# Health Chatbot: Step-by-Step Setup & Build Guide

This guide covers how to build a lightweight, functional Health Chatbot on Telegram using n8n for orchestration and Supabase for reliable data storage. 

The chatbot will support the following core features:
1. Log meals
2. Track the day's logs
3. Edit logged meals
4. Delete logged meals

---

## Step 1: Telegram Bot Setup
1. Open Telegram and search for **@BotFather**.
2. Send the command `/newbot`.
3. Follow the prompts to set a display name and a unique username (must end in `bot`, e.g., `CaloraiHealthBot`).
4. Save the **HTTP API Token** provided by BotFather. You will need this for n8n.
5. (Optional but recommended) Send `/setcommands` to BotFather to create a menu for users:
   * `log - Log a new meal`
   * `track - View today's meals and calories`
   * `edit - Edit a logged meal`
   * `delete - Delete a logged meal`

## Step 2: Supabase Database Setup
1. Create a new project in [Supabase](https://supabase.com/).
2. Navigate to the **SQL Editor** or use the Table Editor to create a `meals` table to store user logs.

Run the following SQL snippet to create your table:
```sql
CREATE TABLE meals (
  id uuid DEFAULT gen_random_uuid() PRIMARY KEY,
  user_id bigint NOT NULL, -- Telegram User ID
  meal_name text NOT NULL,
  calories integer NOT NULL,
  created_at timestamp with time zone DEFAULT now()
);
```
3. Go to **Project Settings > API** in Supabase and copy your **Project URL** and **anon/public key**. You will need these to connect n8n.

## Step 3: n8n Foundation Setup
1. Open your n8n workspace.
2. Go to **Credentials** and add two new credentials:
   * **Telegram API**: Paste the Bot Token from BotFather.
   * **Supabase API**: Paste your Supabase Project URL and Anon API Key.
3. Create a new workflow in n8n.
4. Add a **Telegram Trigger** node. Connect it to your Telegram credential. Set it to listen for `message` updates.

## Step 4: Building the Workflow (Command Routing)
Connect a **Switch** (or **If**) node to the Telegram Trigger to route the logic based on the user's text input.

### 1. Logging a Meal (`/log`)
*   **Trigger Condition**: User sends `/log <meal> <calories>` (e.g., `/log Chicken Salad 450`).
*   **Parsing**: Use a **Code** node to split the message into meal name and calories using regex or basic string operations.
*   **Database Action**: Add a **Supabase** node.
    *   **Operation**: Insert Row.
    *   **Table**: `meals`.
    *   **Fields**: Map `user_id` (from Telegram), `meal_name`, and `calories`.
*   **Response**: Add a **Telegram** node to reply: *"Chicken Salad (450 kcal) logged successfully! ID: <short_id>"*.

### 2. Tracking the Day (`/track`)
*   **Trigger Condition**: User sends `/track`.
*   **Database Action**: Add a **Supabase** node.
    *   **Operation**: Get Many.
    *   **Table**: `meals`.
    *   **Filters**: `user_id = {{$json.message.from.id}}`, and `created_at` > start of today.
*   **Parsing & Response**: Use a Code node to map over the returned rows, formatting them into a readable list (e.g., "1. Chicken Salad - 450 kcal") and summing the total calories. Reply to the user using a **Telegram** node.

### 3. Editing a Meal (`/edit`)
*   **Trigger Condition**: User sends `/edit <uuid> <new_name> <new_calories>`.
*   **Database Action**: Add a **Supabase** node.
    *   **Operation**: Update Row.
    *   **Table**: `meals`.
    *   **Filters**: match `id` with the UUID and `user_id` to ensure users only edit their own meals.
    *   **Update Fields**: `meal_name`, `calories`.
*   **Response**: "Meal updated successfully!"

### 4. Deleting a Meal (`/delete`)
*   **Trigger Condition**: User sends `/delete <uuid>`.
*   **Database Action**: Add a **Supabase** node.
    *   **Operation**: Delete Row.
    *   **Table**: `meals`.
    *   **Filters**: match `id` with the provided UUID and `user_id = {{$json.message.from.id}}`. 
*   **Response**: "Meal deleted successfully!"

---

## Next Steps for Reliability
*   **Error Handling**: Wrap your database and parsing nodes in try/catch logic (or use n8n's error handling paths) so the bot can elegantly reply "Invalid format. Try /log Apple 100" if a user types a bad command.
*   **Conversational Flow (Optional)**: Instead of requesting all variables in one line (`/log apple 100`), you can use a state management approach where `/log` asks "What did you eat?" followed by "How many calories?".