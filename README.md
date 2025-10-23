# Wordie Technical Assessment – Custom WordPress Automation

## Overview

This project implements a **custom WordPress solution** designed to streamline workflows for managing website projects. It introduces a **“Website Projects”** custom post type integrated with **Advanced Custom Fields (ACF)** and **automated webhook notifications**.

When a project is published, the system automatically sends structured data (client name, project status, and title) to a configured webhook endpoint — ensuring **real-time synchronization** with connected systems and eliminating repetitive manual data entry.

---

## Features

* ✅ **Custom Post Type** – Adds a new “Website Projects” section in the WordPress admin.
* ✅ **ACF Integration** – Adds structured project metadata fields (client name, project status, launch date).
* ✅ **Webhook Automation** – Sends a JSON payload to a webhook endpoint on post publication.
* ✅ **Validation & Data Integrity** – Prevents webhook calls until all required fields are filled.
* ✅ **Duplicate Prevention** – Ensures webhook triggers only once per valid publish action.
* ✅ **Real-Time Updates** – Automatically notifies connected systems about project changes.

---

## Tech Stack

* **WordPress (Local WP)**
* **PHP**
* **Advanced Custom Fields (ACF) Plugin**
* **Webhook.site** (for testing webhook delivery)

---

## Installation & Setup

### 1️⃣ Install Local WP

1. Download and install **Local WP** → [https://localwp.com/help-docs/getting-started/installing-local/](https://localwp.com/help-docs/getting-started/installing-local/)

2. Launch Local WP and create a new site:

   * Click **“+ Create a new site”**
   * Name it (e.g., `smart-wordpress-task`)
   * Choose default environment
   * Set up admin credentials (username & password)

3. Once setup is complete, open your site’s Admin via Local (on the image below, click WP Admin) and log in.
   
   <img width="837" height="672" alt="Screenshot 2025-10-23 at 4 06 02 PM" src="https://github.com/user-attachments/assets/21b537e9-0be7-4896-944e-6df4cc85ee26" />


---

### 2️⃣ Install Advanced Custom Fields (ACF)

In **WP Admin → Plugins → Add New**, search for **“Advanced Custom Fields”**, then:

* Click **Install**
* Click **Activate**

You’ll now see an **“ACF”** menu in the sidebar.

<img width="191" height="595" alt="Screenshot 2025-10-23 at 4 08 40 PM" src="https://github.com/user-attachments/assets/0324bc6d-5364-4a1b-9001-8001e00a6bcd" />

---

### 3️⃣ Add the Custom Post Type

1. Open your Local site’s folder:

   * In Local, click the **folder icon** (next to your site name).
     
  <img width="847" height="675" alt="Screenshot 2025-10-23 at 4 11 48 PM" src="https://github.com/user-attachments/assets/d1f141d9-586b-4aee-a727-ba2227eeb6cd" />

   * Navigate to:
     `app/public/wp-content/themes/twentytwentyfive/`
2. Locate and open the **`functions.php`** file with your code editor (VS Code, Cursor, etc.).
3. Paste the following code at the bottom of the file:

```php
/**
 * Smart WordPress Functionality - CPT + Webhook
 */

add_action('init', function () {
    register_post_type('website_project', [
        'label' => 'Website Projects',
        'public' => true,
        'show_in_rest' => true,
        'supports' => ['title', 'editor', 'custom-fields'],
        'menu_icon' => 'dashicons-admin-site-alt3'
    ]);
});
```

This registers a new **“Website Projects”** custom post type that appears in the WordPress Admin sidebar.

---

### 4️⃣ Create Custom Fields with ACF

1. In WP Admin Sidebar → **ACF → Field Groups → Add New**
2. Title the group **“Website Project Fields”**
3. Add the following fields:

| Field Label    | Field Name       | Field Type  | Notes                              |
| -------------- | ---------------- | ----------- | ---------------------------------- |
| Client Name    | `client_name`    | Text        | Required                           |
| Project Status | `project_status` | Select      | Choices: “In Progress”, “Complete” |
| Launch Date    | `launch_date`    | Date Picker | Optional                           |

4. Under **Location Rules**, set:

   ```
   Post Type is equal to Website Project
   ```
   
   <img width="1229" height="343" alt="Screenshot 2025-10-23 at 4 15 30 PM" src="https://github.com/user-attachments/assets/edce6973-4df8-4b0a-aa89-1b2a02f654d0" />


5. Click **Publish**.

---

### 5️⃣ Add Webhook Trigger Logic

Below your CPT registration in the same `functions.php`, add:

```php
/**
 * Helper class to send webhook
 */
class Wordie_Automation_Helper {
    public static function send_webhook($url, $payload) {
        $response = wp_remote_post($url, [
            'headers' => ['Content-Type' => 'application/json'],
            'body' => wp_json_encode($payload),
        ]);

        if (is_wp_error($response)) {
            error_log('Webhook failed: ' . $response->get_error_message());
        }
    }
}

/**
 * Trigger webhook when Website Project is published
 */
add_action('save_post', function ($post_id) {
    // Only for website_project post type
    if (get_post_type($post_id) !== 'website_project') {
        return;
    }
    
    // Only when post is published (not draft, auto-draft, etc.)
    if (get_post_status($post_id) !== 'publish') {
        return;
    }
    
    // Skip autosaves and revisions
    if (wp_is_post_autosave($post_id) || wp_is_post_revision($post_id)) {
        return;
    }
    
    // Get post data
    $title = get_the_title($post_id);
    $client_name = get_field('client_name', $post_id);
    $project_status = get_field('project_status', $post_id);
    
    // Only send webhook if we have the ACF field data (avoid first call with null values)
    if (empty($client_name) && empty($project_status)) {
        return;
    }

    $payload = [
        'title' => $title,
        'client_name' => $client_name,
        'status' => $project_status
    ];

    // Replace with your real webhook.site URL
    Wordie_Automation_Helper::send_webhook('https://webhook.site/14043fbc-af23-4bfd-a1ee-4c685909be89', $payload);
});
```

---

### 6️⃣ Test the Automation

1. In WP Admin → **Website Projects → Add Post**
2. Fill in:

   * **Title**
   * **Client Name**
   * **Project Status**
   * **Launch Date**
     
  <img width="1153" height="901" alt="Screenshot 2025-10-23 at 4 19 43 PM" src="https://github.com/user-attachments/assets/b691c019-9288-4faf-8584-1114199301ea" />

3. Click **Publish**
4. Visit your **Webhook.site URL** — you should see a POST request with a JSON payload like:

```json
{
  "title": "Galderma",
  "client_name": "Galderma Company",
  "status": "In Progress"
}
```

<img width="1437" height="587" alt="Screenshot 2025-10-23 at 4 20 19 PM" src="https://github.com/user-attachments/assets/9ac791b2-12a9-4c0c-af28-b77e51d15204" />

---

## System Behavior Summary

| Action                          | Description                                    |
| ------------------------------- | ---------------------------------------------- |
| **Save draft**                  | Webhook not triggered                          |
| **Publish without ACF fields**  | Webhook not triggered                          |
| **Publish with valid ACF data** | Webhook triggered once                         |
| **Update post**                 | Webhook re-sends only when status/data changes |

---

## AI Assistance Reflection

The **AI assistant** was instrumental in:

* Suggesting efficient WordPress hook usage (`save_post`)
* Debugging ACF timing issues
* Preventing duplicate webhook calls
* Structuring payload validation
* Optimizing automation for cleaner maintainability

This project was a great example of how I adapt and learn fast when faced with something new. With **Cursor (AI)** as my coding partner, I was able to quickly grasp how WordPress hooks and ACF work, refine the logic, and build a smooth webhook automation flow. It really showed how I use my technical foundation, curiosity, and AI tools together to figure things out and deliver results — even in areas outside my everyday stack.

---
