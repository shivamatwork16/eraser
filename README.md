<p><a target="_blank" href="https://app.eraser.io/workspace/1NALc1v7gpA7ndUolVWu" id="edit-in-eraser-github-link"><img alt="Edit in Eraser" src="https://firebasestorage.googleapis.com/v0/b/second-petal-295822.appspot.com/o/images%2Fgithub%2FOpen%20in%20Eraser.svg?alt=media&amp;token=968381c8-a7e7-472a-8ed6-4a6626da5501"></a></p>

# eraser
```

```
```
async function sendReviewReminderCsvOne() {
  try {
    const reviewCsvs = await db.reviewinvitecsv.find({
      "users.first_sent": { $ne: null }, // Only process users who have received the first email
      "users.second_sent": null, // Only process users who haven't received the second email
      reminder: true, // Only process those that have reminders set to true.
    });

    for await (const reviewCsv of reviewCsvs) {
      const pageSetting = await db.reviewpagesetting.findById(
        reviewCsv.pageSettingId,
      );

      if (!pageSetting) {
        console.warn(`Page setting not found for reviewCsv ${reviewCsv._id}`);
        continue;
      }

      if (
        !pageSetting.reminder_email_invite_template ||
        !pageSetting.invite_reminder_after_firstemail
      ) {
        console.warn(
          `Reminder template or first reminder delay not set for page setting ${pageSetting._id}`,
        );
        continue;
      }

      const usersToSendReminder = reviewCsv.users.filter((user) => {
        if (!user.first_sent || user.second_sent) {
          return false; // Skip users without first_sent or with second_sent
        }

        const timeSinceFirstSent = Date.now() - user.first_sent.getTime();
        const reminderDelay =
          pageSetting.invite_reminder_after_firstemail * 24 * 60 * 60 * 1000; // Convert days to milliseconds

        return timeSinceFirstSent >= reminderDelay;
      });

      for await (const user of usersToSendReminder) {
        const emailpayload = {
          templateId: pageSetting.reminder_email_invite_template,
          email: user.email,
          smtpId: pageSetting.reviewInviteSmtpId,
          addedBy: pageSetting.addedBy,
          logo: pageSetting.logo,
        };

        try {
          await ReviewEmails.reviewCsvInvite(emailpayload);
          user.second_sent = new Date();
        } catch (emailError) {
          console.error(
            `Error sending reminder 1 to ${user.email}:`,
            emailError,
          );
        }
      }

      await reviewCsv.save();
    }
  } catch (err) {
    console.error("Error sending review reminder 1:", err);
  }
}

async function sendReviewReminderCsvTwo() {
  try {
    const reviewCsvs = await db.reviewinvitecsv.find({
      "users.second_sent": { $ne: null }, // Only process users who have received the second email
      "users.last_sent": null, // Only process users who haven't received the last email
      reminder: true, // Only process those that have reminders set to true.
    });

    for await (const reviewCsv of reviewCsvs) {
      const pageSetting = await db.reviewpagesetting.findById(
        reviewCsv.pageSettingId,
      );

      if (!pageSetting) {
        console.warn(`Page setting not found for reviewCsv ${reviewCsv._id}`);
        continue;
      }

      if (
        !pageSetting.reminder_email_invite_template_last ||
        !pageSetting.last_invite_reminder_after_secondemail
      ) {
        console.warn(
          `Last reminder template or last reminder delay not set for page setting ${pageSetting._id}`,
        );
        continue;
      }

      const usersToSendReminder = reviewCsv.users.filter((user) => {
        if (!user.second_sent || user.last_sent) {
          return false; // Skip users without second_sent or with last_sent
        }

        const timeSinceSecondSent = Date.now() - user.second_sent.getTime();
        const reminderDelay =
          pageSetting.last_invite_reminder_after_secondemail *
          24 *
          60 *
          60 *
          1000; // Convert days to milliseconds

        return timeSinceSecondSent >= reminderDelay;
      });

      for await (const user of usersToSendReminder) {
        const emailpayload = {
          templateId: pageSetting.reminder_email_invite_template_last,
          email: user.email,
          smtpId: pageSetting.reviewInviteSmtpId,
          addedBy: pageSetting.addedBy,
          logo: pageSetting.logo,
        };

        try {
          await ReviewEmails.reviewCsvInvite(emailpayload);
          user.last_sent = new Date();
        } catch (emailError) {
          console.error(
            `Error sending reminder 2 to ${user.email}:`,
            emailError,
          );
        }
      }

      await reviewCsv.save();
    }
  } catch (err) {
    console.error("Error sending review reminder 2:", err);
  }
}
```




<!--- Eraser file: https://app.eraser.io/workspace/1NALc1v7gpA7ndUolVWu --->