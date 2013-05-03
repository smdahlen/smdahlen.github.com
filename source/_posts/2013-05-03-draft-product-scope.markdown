---
layout: post
title: "Draft product scope"
date: 2013-05-03 15:27
comments: true
keywords: pomodoro,tasks,productivity,gamification,marinara
description: Establishes the product scope for Marinara, a time management tool, based on the Pomodoro Technique.
---

With an [operational approach][1] for managing my environment established, I
shifted my attention toward drafting product scope over the past few weeks.
This post outlines the core product features and acceptance criteria that will
be referenced during the upcoming design and development activities.

<!-- more -->

As I mentioned in my second [post][2], I will be implementing a task management
product that provides first-class support for the [Pomodoro Technique][3]. One
of the aspects of the technique that I find intriguing is the use of data to
review progress and provide motivation for continued success. For those who
have dabbled with personal productivity solutions over the years, it would come
as no surprise that sticking with a solution is hard. Without continued use,
positive habits never form and the expected productivity gains are never
realized. With my product, I intend to leverage the data collected from the
Pomodoro Technique to create a positive feedback loop motivating the individual
to modify their behavior over time.

To begin, I established the product goal for Marinara (the working name) as a
tool to increase personal productivity and assist in managing expectations
(with yourself, boss, or client).

Next, I crisply defined the core high-level features that would support the
product goal:

1. A proven **workflow** based on the Pomodoro Technique that enhances your focus
   and concentration to complete complex, creative projects.
2. Challenging **gameplay** that offers a sustainable path towards mastery of
   the technique.
3. In-depth **statistics and charts** that increases awareness of your actions
   to support effective decision making.

Finally, I brainstormed, drafted, and revised a set of user stories and
acceptance criteria supporting the high-level features. Much time was given to
both creating a set of compelling game elements and ensuring testability of the
entire scope. The scope is succinct and intended as a foundation for creating
the user experience design -- little attention was paid to drafting copy for
end users either as a manual or marketing material.

Without much further adieu here is the scope for Marinara -- *a product that
helps you tackle your creative project and have fun doing it*.

- - -

Pomodoro
--------
- As a player, I want to start a pomodoro for a specific task.
    - It should display a timer counting down from 25 minutes.
    - It should display a timer counting down from 5 minutes when the 25 minute
      count down is complete.
    - It should prompt the player for an estimate if one has not been provided
      and the player is at the intermediate level or above.
- As a player, I want to cancel the pomodoro.
    - It should clear the timer count down.
    - It should record the cancelled pomodoro.
- As a player, I want to be notified when the pomodoro completes.
    - It should display a notification when the 25 minute count down is
      complete.
    - It should play an audio notification when the 25 minute count down is
      complete.
    - It should display a notification when the 5 minute count down is
      complete.
    - It should play an audio notification when the 5 minute count down is
      complete.
    - It should display a notification to start the next pomodoro within 60
      seconds to continue a set if a player is at the advanced beginner level
      or above.
    - It should record a completed pomodoro when the 25 minute count down is
      complete.
    - It should disable changing an estimate after a completed pomodoro has
      been recorded for a task.
- As a player, I want to record an internal interruption.
    - It should record an internal interruption associated to the current
      pomodoro.
- As a player, I want to record an external interruption.
    - It should record an external interruption associated to the current
      pomodoro.
- As a player, I want to view the number of pomodoros completed for the
  current set.
    - It should unlock this feature when a player completes the beginner level.
    - It should display the number of pomodoros completed out of 4 for the
      set.
    - It should reset the current set to 0 if the next pomodoro is not started
      within 60 seconds after the previous pomodoro ended.
    - It should display a notification to take a 15 to 30 minute break when a
      set is completed.

Today Sheet
-----------
- As a player, I want to view the tasks for today.
    - It should display a section for planned tasks.
    - It should display a section for unplanned / urgent tasks.
    - It should display the title and pomodoros completed for a task.
    - It should display the estimate for a task if the player is at the
      intermediate level or above.
    - It should clear all tasks from yesterday.
- As a player, I want to view a set of recommended tasks to work on today.
    - It should display yesterday's unfinished tasks.
    - It should display tasks due within the next few days.
    - It should display tasks related to other recently completed tasks.
- As a player, I want to add a new task to the today sheet.
    - It should associate the new task to the player's default inventory.
- As a player, I want to remove a task from the today sheet.
    - It should clear the task from the today sheet.
- As a player, I want to reorder tasks on the today sheet.
    - It should support drag and drop ordering.
    - It should save the ordering of tasks on the today sheet.
- As a player, I want to commit to planned tasks for today.
    - It should unlock this feature when a player completes the intermediate
      level.
    - It should require a minimum of 10 pomodoros of effort.
    - It should disable new tasks from be added to the planned section after
      a player commits.
    - It should record a commitment met if all planned tasks are completed
      by the end of the day.
    - It should display a notification when the commitment is met for today.
- As a player, I want to view a recommended amount of planned tasks to work
  on today.
    - It should unlock this feature when a player completes the advanced
      beginner level.
    - It should display the pomodoro effort for the planned tasks.
    - It should display the moving average for planned pomodoros completed.
    - It should display a warning if the planned effort is 20% greater than
      the moving average.
- As a player, I want to mark a task as complete.
    - It should strikeout the task.
    - It should update the status of the task to complete.
- As a player, I want to view a summary of today's progress.
    - It should display the number of pomodoros completed.
    - It should display the number of interruptions recorded.
    - It should display the points earned.
    - It should display the number of sets completed if the player is at the
      advanced beginner level or above.
    - It should display the estimation accuracy if the player is at the
      intermediate level or above.

Accounts
--------
- As a player, I want to sign up for the service using my email address.
    - It should require a full name.
    - It should require an email address.
    - It should require a complex password.
    - It should confirm the complex password.
    - It should accept a time zone.
- As a player, I want to verify my identity with an email confirmation.
    - It should set the account status to pending verification upon sign up.
    - It should display a notification to verify the email address upon sign up.
    - It should send an email containing an activation link to the new player
      that expires in 24 hours upon sign up.
    - It should set the account status to active when the link is visited.
    - It should display a warning when attempting to login with an account
      status set to pending verification.
    - It should support the player requesting a new email containing an
      activation link.
- As a player, I want to reset my password that I have forgotten.
    - It should require an email address.
    - It should send an email containing a reset link that expires in 24 hours.
    - It should require a complex password after visiting the reset link.
    - It should confirm the complex password.
    - It should update the player's password if it is confirmed.
- As a player, I want to login with my email address.
    - It should require an email address.
    - It should require a password.
    - It should display a warning if the credentials were incorrect.
    - It should redirect to the player's home page if correct.
- As a player, I want to manage my account settings.
    - It should support updating the player's timezone.
    - It should support updating the player's email address.
    - It should support updating the player's password.
    - It should support updating the player's full name.
    - It should support toggling audio notifications.
- As a player, I want to login with my Facebook credentials.
    - It should request permission to access the player's public profile.
    - It should request permission to access the player's email address.
    - It should create a new account with the player's Facebook token if it
      does not exist.
- As a player, I want to purchase a subscription.
    - It should support purchase of a monthly subscription.
    - It should support purchase of a yearly subscription.
    - It should accept credit cards and PayPal directly on the service website.
    - It should send an email containing an invoice for the subscription.
- As a player, I want to apply a coupon to receive a subscription discount.
    - It should require a valid coupon code.
    - It should apply the associated coupon code discount to the selected
      subscription.
- As the service owner, I want an account locked until a subscription is
  purchased.
    - It should set the account status to locked when the player completes the
      beginner level.
    - It should set the account status to locked when a subscription lapses.
    - It should redirect locked accounts to purchase a subscription.

Inventory Sheets
----------------
- As a player, I want to create an new inventory sheet.
    - It should require a name.
    - It should support setting the inventory as the player's default.
- As a player, I want to remove an inventory sheet.
    - It should prompt the player to confirm removing all tasks in the
      inventory.
    - It should remove all tasks in the inventory.
- As a player, I want to view tasks in an inventory.
    - It should support sorting tasks by due date.
    - It should support sorting tasks by estimate if a player is at the
      intermediate level or above.
    - It should support toggling between complete and incomplete tasks.
    - It should display incomplete tasks by default.
    - It should display tasks sorted by due date by default.
    - It should display an indicator if the estimate is greater than 7 if the
      player is at the intermediate level or above.
- As a player, I want to add a new task to an inventory.
    - It should require a title.
    - It should accept an optional due date.
    - It should accept an optional estimate if the player is at the
      intermediate level or above.
    - It should accept zero or more tags.
    - It should accept a note.
    - It should support associating the task to a parent task.
- As a player, I want to edit an existing task.
    - It should update the title.
    - It should update the due date.
    - It should update the estimate if the player is at the intermediate level
      or above.
    - It should update the tags.
    - It should update the note.
- As a player, I want to remove a task from an inventory.
    - It should soft delete the task if it has pomodoros associated to it.
    - It should hard delete the task if it has no pomodoros associated to it.
- As a player, I want to move a task to another inventory.
    - It should clear the task from the current inventory.
    - It should update the task to the specified inventory.
- As a player, I want to select tasks to work on today.
    - It should display the selected tasks on the planned section for today.
    - It should display an indicator next to tasks in an inventory that are
      planned for today.
- As a player, I want to deselect tasks to work on today.
    - It should clear the deselected tasks from the planned section for today.
- As a player, I want to filter tasks by tag.
    - It should support the player selecting zero or more tags to filter tasks.
    - It should display tasks that contain one of the selected tags.
- As a player, I want to bundle small tasks into a larger one.
    - It should support the player selecting two or more tasks.
    - It should create a new parent task for the selected tasks.

Metrics
-------
- As a player, I want to view a metric for pomodoros completed per day.
    - It should equal the 14-day moving average for pomodoros completed per day.
    - It should not include inactive days.
    - It should equal N/A if there is less than 8 active days within the last
      14 days.
    - It should display the component metrics of unplanned and planned pomodoros
      completed per day.
- As a player, I want to view a metric for pomodoros cancelled per day.
    - It should equal the 14-day moving average for pomodoros cancelled per day.
    - It should not include inactive days.
    - It should equal N/A if there is less than 8 active days within the last
      14 days.
- As a player, I want to view a metric for sets completed per day.
    - It should unlock this feature when the player completes the beginner
      level.
    - It should equal the 14-day moving average for sets completed per day.
    - It should not include inactive days.
    - It should equal N/A if there is less than 8 active days within the last
      14 days.
- As a player, I want to view a metric for total interruptions per day.
    - It should equal the 14-day moving average for total interruptions per day.
    - It should not include inactive days.
    - It should equal N/A if there is less than 8 active days within the last
      14 days.
    - It should display the component metrics of internal and external
      interruptions per day.
- As a player, I want to view a metric for estimation accuracy.
    - It should unlock this feature when the player completes the advanced
      beginner level.
    - It should equal the 14-day moving average for estimation accuracy per day.
    - It should not include inactive days.
    - It should equal N/A if there is less than 8 active days within the last
      14 days.
- As a player, I want to view a metric for estimation bias.
    - It should unlock this feature when the player completes the advanced
      beginner level.
    - It should equal the 14-day moving average of the daily difference between
      estimated and actual pomodoros.
    - It should not include inactive days.
    - It should equal N/A if there is less than 8 active days within the last
      14 days.
- As a player, I want to view a metric for the current streak of commitments
  met.
    - It should unlock this feature when the player completes the intermediate
      level.
    - It should determine a commitment met if all planned tasks are completed
      and the total effort was 10 or more pomodoros.
    - It should equal the current continuous days of commitments met.
    - It should not include inactive days.
    - It should reset if there is less than 8 active days within the last
      14 days.
- As a player, I want to view a metric for the longest streak of commitments
  met.
    - It should unlock this feature when the player completes the intermediate
      level.
    - It should equal the longest streak of commitments met since the
      player started.

Charts
------
- As a player, I want to view a chart visualizing the pomodoros completed per
  day.
    - It should display a segmented bar chart with completed and cancelled
      pomodoros per day.
    - It should display a stacked bar for unplanned and planned
      completed pomodoros per day.
    - It should display a trend line with the 14-day moving average of
      completed pomodoros per day.
    - It should display the last 60 days of data.
    - It should not include inactive days.
- As a player, I want to view a chart visualizing the total interruptions per
  day.
    - It should display a stacked bar chart with internal and external
      interruptions per day.
    - It should display a trend line with the 14-day moving average of total
      interruptions per day.
    - It should display the last 60 days of data.
    - It should not include inactive days.
- As a player, I want to view a chart visualizing the estimation accuracy.
    - It should unlock this feature when the player completes the advanced
      beginner level.
    - It should display a bar chart with estimation accuracy per day.
    - It should display a trend line with the 14-day moving average of
      estimation accuracy.
    - It should display the last 60 days of data.
    - It should not include inactive days.
- As a player, I want to view a chart visualizing pomodoros completed per tag.
    - It should display a pie chart with the top 5 tags collapsing the
      remaining tags into an 'other' category.
    - It should include the last 14 active days of data.

Progress Reports
----------------
- As a player, I want to receive a weekly progress report via email.
    - It should include upcoming tasks due within the next 14 days.
    - It should include tasks completed over the past 7 days.
    - It should include the points earned over the past 7 days.
    - It should include the leaderboard rank.
    - It should include the percent change for pomodoros completed per day rate
      over the past 7 days.
    - It should include the percent change for estimation accuracy over the
      past 7 days if the player is at the intermediate level or above.
    - It should include the progress of the current achievements.

Points
------
- As a player, I want to earn points for completing a pomodoro.
    - It should award 5 points.
- As a player, I want to receive points for completing a set.
    - It should unlock this feature when the player completes the beginner
      level.
    - It should award 5 points.
- As a player, I want to earn points for correctly estimating a task.
    - It should unlock this feature when the player completes the advanced
      beginner level.
    - It should award points equal to the estimate of the task.
- As a player, I want to earn points for meeting my commitment for today.
    - It should unlock this feature when the player completes the intermediate
      level.
    - It should award points equal to committed pomodoros for today.

Beginner Level
--------------
- As a player, I want to read an introduction to the beginner level.
    - It should display content introducing the beginner level objective.
    - It should display content introducing the timer and inventory features.
    - It should display content introducing the beginner level achievements.
- As a player, I want to view my progress completing the beginner level
  achievements.
    - It should display the number of times a player has completed 8 pomodoros
      in one day.
    - It should display the current rate of pomodoros per day.
    - It should display the number of recorded interruptions.
- As a player, I want to be notified when completing a beginner level
  achievement.
    - It should display a notification when a player completes 8 pomodoros in
      one day 3 times.
    - It should display a notification when a player achieves a rate of 4
      pomodoros per day.
    - It should display a notification when a player records 50 interruptions.
- As a player, I want to be notified when completing the beginner level.
    - It should display a notification when a player completes all three
      achievements and accumulates 300 points.
    - It should advance the player to the advanced beginner level.

Advanced Beginner Level
-----------------------
- As a player, I want to read an introduction to the advanced beginner level.
    - It should display content introducing the advanced beginner level
      objective.
    - It should display content introducing the timer set feature.
    - It should display content introducing the advanced beginner achievements.
- As a player, I want to view my progress completing the advanced beginner level
  achievements.
    - It should display the number of times a player has completed a set.
    - It should display the current rate of pomodoros per day.
    - It should display the percent improvement of the interruption rate and
      cancelled pomodoros.
- As a player, I want to be notified when completing an advanced beginner level
  achievement.
    - It should display a notification when a player completes 20 sets.
    - It should display a notification when a player achieves a rate of 8
      pomodoros per day.
    - It should display a notification when a player decreases the interruption
      rate and cancelled pomodoros by 50%.
- As a player, I want to be notified when completing the advanced beginner
  level.
    - It should display a notification when a player completes all three
      achievements and accumulates 1500 points.
    - It should advance the player to the intermediate level.

Intermediate Level
------------------
- As a player, I want to read an introduction to the intermediate level.
    - It should display content introducing the intermediate level objective.
    - It should display content introducing the estimation feature.
    - It should display content introducing the intermediate achievements.
- As a player, I want to view my progress completing the intermediate level
  achievements.
    - It should display the number of times a player achieves a 90% estimation
      accuracy in one day.
    - It should display the current rate of sets completed per day.
    - It should display the percent improvement of unplanned pomodoros
      completed.
- As a player, I want to be notified when completing an intermediate level
  achievement.
    - It should display a notification when a player achieves a 90% estimation
      accuracy in one day 20 times.
    - It should display a notification when a player achieves a rate of 2 sets
      completed per day.
    - It should display a notification when a player decreases unplanned
      pomodoros completed by 50%.
- As a player, I want to be notified when completing the intermediate level.
    - It should display a notification when a player completes all three
      achievements and accumulates 5000 points.
    - It should advance the player to the advanced level.

Advanced Level
--------------
- As a player, I want to read an introduction to the advanced level.
    - It should display content introducing the advanced level objective.
    - It should display content introducing the daily commitment feature.
    - It should display content introducing the advanced achievements.
- As a player, I want to view my progress completing the advanced level
  achievements.
    - It should display the number of times a player has met his daily
      commitment.
    - It should display the current estimation accuracy rate.
    - It should display the percent improvement of estimation bias.
- As a player, I want to be notified when completing an advanced level
  achievement.
    - It should display a notification when a player meets his daily commitment
      20 times.
    - It should display a notification when a player achieves a estimation
      accuracy rate of 90%.
    - It should display a notification when a player reduces his estimation bias
      by 50%.
- As a player, I want to be notified when completing the advanced level.
    - It should display a notification when a player completes all three
      achievements and accumulates 15000 points.
    - It should advance the player to the master level.

Master Level
------------
- As a player, I want to read an introduction the master level.
    - It should display content introducing the master level objective.
    - It should display content introducing the master achievement.
- As a player, I want to view my progress completing the master level
  achievements.
    - It should display the longest commitment streak.
- As a player, I want to be notified when completing a master level
  achievement.
    - It should display a notification when a player achieves a commitment
      streak of 20 days.

Supporters
----------
- As a player, I want to search for other players.
    - It should support searching by name.
    - It should support searching by email address.
- As a player, I want to request support from another player.
    - It should display a support request for the receiver.
    - It should send an email to the receiver that support has been requested.
- As a player, I want to accept a support request from another player.
    - It should support accepting a support request.
    - It should support ignoring a support request.
    - It should create a support relationship when the request is accepted.
    - It should send an email to the requestor that the support request has
      been accepted.
- As a player, I want to remove support for another player.
    - It should delete the support relationship.
- As a player, I want to view a leaderboard of my supporters.
    - It should rank supporters and the player by points earned for today.
    - It should rank supporters and the player by points earned for the past
      14 days.
    - It should rank supporters and the player by points earned for all time.
    - It should support viewing key metrics for each supporter.
- As a player, I want to invite colleagues from LinkedIn to support me.
    - It should support linking to the player's LinkedIn account.
    - It should display the player's LinkedIn connections.
    - It should support selecting one or more connections to invite.
    - It should create a support request for connections that have an account.
    - It should send an email invitation to the service for connections that do
      not have an account.
- As a player, I want to invite family and friends from Facebook to support me.
    - It should support linking to the player's Facebook account.
    - It should display the player's Facebook friends.
    - It should support selecting one or more friends to invite.
    - It should create a support request for friends that have an account.
    - It should send an email invitation to the service for friends that do
      not have an account.

Future Features
---------------
- keyboard shortcuts
- email tasks
- shared inventories
- advanced search/filtering of tasks
- task comments
- integration with issue tracking and project management services
- organization volume discounts
- cooperative and head-to-head gameplay


[1]: http://shawn.dahlen.me/blog/2013/04/12/manage-all-application-environments-with-vagrant/
[2]: http://shawn.dahlen.me/blog/2013/03/05/prepare-business-operations/
[3]: http://www.pomodorotechnique.com/
