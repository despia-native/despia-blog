---
title: "Building Your First Native Mobile App with Lovable.dev"
seoTitle: "Building Your First Native Mobile App with Lovable.dev"
seoDescription: "Learn how to build and deploy native iOS and Android apps in 30 minutes using Lovable's AI-powered development and Despia's one-click publishing."
datePublished: 2025-09-04T01:37:30.967Z
cuid: cmf4qktav000202jlef18boa7
slug: building-your-first-native-mobile-app-with-lovabledev
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1756949661655/dd76b9ee-c7e7-4a55-98fb-804ec30f1fa1.png
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1756949743403/b8f94725-5659-4919-953d-dec312aacba9.png
tags: lovable, vibe-coding, lovable-ai, despia

---

A lot of developers have been asking about the fastest way to get started with Despia. Here's a workflow that's been working really well: using Lovable.dev to build your React app, then bringing it into Despia for native deployment.

## The Setup

Lovable.dev is an AI-powered development environment that generates React applications from natural language. It's particularly useful for Despia because it outputs clean, production-ready React code that integrates perfectly with our SDK.

Here's the basic workflow we'll walk through:

## Start with Your Idea

Go to Lovable.dev and describe what you want to build. Be specific about features. For this example, let's build a habit tracker:

*"Build a habit tracking app with daily check-ins, streak counters, and statistics. Include a clean interface with cards for each habit."*

Lovable generates the entire React application instantly. You get components, routing, state management - everything's already wired up.

## Integrate the Despia SDK

Once you have your base app, add the Despia SDK to enable native features:

```bash
npm install despia-native
```

Now you can enhance your app with native capabilities. Here's what that looks like in practice:

```javascript
import despia from 'despia-native';

// When user completes a habit
const markHabitComplete = async (habitId) => {
  // Native haptic feedback
  await despia('successhaptic://');
  
  // Update your state
  setHabits(prev => ...);
  
  // Send local notification for tomorrow
  await despia('sendlocalpushmsg://push.send?s=86400&msg=Time for your daily habits!');
};
```

## Common Integration Points

Based on what we've seen from developers, here are the most useful native features to add to a Lovable-generated app:

### Haptic Feedback

Add tactile responses to important interactions:

```javascript
// Button presses
await despia('lighthaptic://');

// Success states
await despia('successhaptic://');

// Errors or warnings
await despia('errorhaptic://');
```

### Local Notifications

For scheduled notifications without a backend:

```javascript
const scheduleReminder = async (seconds, message) => {
  await despia(`sendlocalpushmsg://push.send?s=${seconds}&msg=${message}`);
};
```

Despia includes OneSignal integration for push notifications. Here's how to connect users to your backend:

```javascript
// Get the OneSignal player ID from Despia
const setupPushNotifications = async () => {
  // Register for push notifications
  await despia('registerpush://');
  
  // Get OneSignal player ID
  const data = await despia('getonesignalplayerid://', ['onesignalplayerid']);
  const playerId = data.onesignalplayerid;
  
  // Send to your backend to link with user
  await fetch('https://api.yourapp.com/users/link-onesignal', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: getCurrentUserId(),
      oneSignalPlayerId: playerId
    })
  });
};
```

On your backend, store this relationship:

```sql
-- Simple linking table
CREATE TABLE user_push_tokens (
  user_id VARCHAR(255),
  onesignal_player_id VARCHAR(255),
  created_at TIMESTAMP
);
```

Now you can send targeted push notifications from your backend:

```javascript
// Backend API example - send notification to specific user
const sendPushToUser = async (userId, message) => {
  // Get player ID from your database
  const playerId = await db.query(
    'SELECT onesignal_player_id FROM user_push_tokens WHERE user_id = ?',
    [userId]
  );
  
  // Send via OneSignal API
  await fetch('https://onesignal.com/api/v1/notifications', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': 'Basic YOUR_ONESIGNAL_API_KEY'
    },
    body: JSON.stringify({
      app_id: 'YOUR_ONESIGNAL_APP_ID',
      include_player_ids: [playerId],
      contents: { en: message },
      headings: { en: 'New Update' }
    })
  });
};
```

### Local Notifications

For scheduled notifications without a backend:

```javascript
const scheduleReminder = async (seconds, message) => {
  await despia(`sendlocalpushmsg://push.send?s=${seconds}&msg=${message}`);
};
```

### In-App Purchases

Monetize with RevenueCat integration:

```javascript
const upgradeToPro = async () => {
  const userId = getUserId();
  await despia(`revenuecat://purchase?external_id=${userId}&product=pro_monthly`);
};
```

## Testing Your Integration

The Despia simulator shows you exactly how your app will behave on actual devices. This is crucial for testing haptics, notifications, and other native features that don't work in a browser.

A helpful pattern is to check if you're running in Despia:

```javascript
const isDespia = navigator.userAgent.includes('despia');

const saveData = async (data) => {
  if (isDespia) {
    await despia('lighthaptic://');
  }
  // Continue with save logic
};
```

## Performance Considerations

When combining Lovable output with Despia, keep these points in mind:

1. **State Management** - Lovable typically uses React state. This works perfectly with Despia's native runtime.
    
2. **Asset Optimization** - Lovable generates optimized images, but consider using native caching:
    
    ```javascript
    await despia(`savethisimage://?url=${imageUrl}`);
    ```
    
3. **Navigation** - Lovable's React Router works seamlessly. Add haptics to route changes for better UX:
    
    ```javascript
    const navigate = (path) => {
      despia('lighthaptic://');
      history.push(path);
    };
    ```
    

## Deployment Options

After integrating Despia features into your Lovable app, you have two powerful deployment options:

### Option 1: Direct URL Link (Recommended for Speed)

Simply link your Lovable app URL directly to Despia:

1. Get your Lovable app URL (e.g., `https://your-app.lovable.app`)
    
2. Paste it into [Despia.com](https://despia.com/)
    
3. Configure your native settings
    
4. Click "Publish"
    

Your app now enjoys **over-the-air updates** - any changes you make in Lovable instantly appear in your published app without resubmitting to app stores.

### Option 2: Self-Hosted URL

If you prefer to self-host:

1. Export from Lovable and deploy to your own hosting
    
2. Link your live URL to Despia (e.g., `https://your-domain.com`)
    
3. Configure and publish
    

This gives you complete control while still enabling instant OTA updates whenever you update your hosted code.

The entire process typically takes about 30 minutes. Despia automates the entire deployment pipeline, so you don't need Xcode, Android Studio, or any knowledge of app store submission processes.

## Real Example: Task Manager

Here's a complete component showing Lovable + Despia integration:

```javascript
import React, { useState } from 'react';
import despia from 'despia-native';

function TaskList() {
  const [tasks, setTasks] = useState([]);
  
  const addTask = async (task) => {
    // Haptic feedback
    await despia('lighthaptic://');
    
    // Add to state
    setTasks([...tasks, task]);
    
    // Schedule reminder
    if (task.reminder) {
      await despia(`sendlocalpushmsg://push.send?s=3600&msg=Task due: ${task.name}`);
    }
  };
  
  const completeTask = async (id) => {
    // Success feedback
    await despia('successhaptic://');
    
    // Update task
    setTasks(tasks.map(t => 
      t.id === id ? {...t, complete: true} : t
    ));
    
    // Take screenshot for sharing
    await despia('takescreenshot://');
  };
  
  return (
    <div>
      {tasks.map(task => (
        <TaskCard 
          key={task.id}
          task={task}
          onComplete={() => completeTask(task.id)}
        />
      ))}
    </div>
  );
}
```

## Common Patterns

Through working with developers, we've identified patterns that work well:

**Progressive Enhancement** - Start with core functionality in Lovable, then layer on native features where they add value.

**Feature Detection** - Always check if running in Despia before calling native APIs.

**User Feedback** - Use haptics liberally. Users expect tactile feedback in native apps.

## Debugging Tips

When something isn't working:

1. Check the Despia console for any SDK errors
    
2. Verify you're using the correct command format: `feature://action?params`
    
3. Test in the Despia simulator, not just your browser
    
4. Make sure the SDK is properly imported
    

## Next Steps

Once you're comfortable with the basic integration:

* Explore widgets for home screen presence
    
* Implement background location for fitness apps
    
* Add Siri shortcuts for quick actions
    
* Set up push notifications with OneSignal
    

The combination of Lovable's rapid development and Despia's native capabilities means you can build production apps faster than ever. The key is starting simple and progressively adding native features where they enhance the user experience.

## Resources

* [Despia SDK Documentation](https://docs.despia.com/)
    
* [NPM Package Documentation](https://www.npmjs.com/package/despia-native)
    
* [Lovable.dev](https://lovable.dev/)
    

Have questions about integrating Lovable with Despia? Check out our docs at [docs.despia.com](https://docs.despia.com/) or reach out to our team at [support@despia.com](mailto:support@despia.com)