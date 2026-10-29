State Synchronization Bug — Explained with Code
What it means
A state synchronization bug happens when your data (state) says one thing, but your UI shows another — because you forgot to manually update every place that displays that data.

In vanilla JS, you are the synchronizer. If you miss one spot, the UI lies.

A realistic example: Doctor verification
Imagine a hospital app. A doctor can be verified or not.

The same doctor’s status appears in 3 places:

Doctor’s profile card

Active queue list

Appointment modal

The state
js
const doctor = {
  id: 1,
  name: "Dr. Mehta",
  isVerified: false,
};
The HTML
html
<!-- Profile card -->
<div id="profile-card">
  <h2>Dr. Mehta</h2>
  <span id="profile-badge">Not Verified</span>
</div>

<!-- Active queue -->
<ul id="queue">
  <li>
    Dr. Mehta
    <span id="queue-badge">Not Verified</span>
  </li>
</ul>

<!-- Appointment modal -->
<div id="modal">
  <p>Dr. Mehta</p>
  <span id="modal-badge">Not Verified</span>
</div>
The bug: you update the data, but forget one badge
Somewhere in your code, the admin verifies the doctor:

js
function verifyDoctor() {
  // 1. Update the state
  doctor.isVerified = true;

  // 2. Manually update the UI... but only SOME of it
  const profileBadge = document.getElementById('profile-badge');
  profileBadge.innerText = 'Verified';
  profileBadge.classList.add('bg-green-500');

  const queueBadge = document.getElementById('queue-badge');
  queueBadge.innerText = 'Verified';
  queueBadge.classList.add('bg-green-500');

  // ❌ Forgot the modal badge!
}
What the user sees
Place	Data says	UI shows
Profile card	Verified	✅ Verified
Queue	Verified	✅ Verified
Appointment modal	Verified	❌ Not Verified
The modal is now out of sync with the state.

This is the bug: the data is correct, but one part of the UI is stale.

Why it gets worse
Now imagine the doctor status is updated from multiple places:

Admin verifies from the profile page

Backend sends a websocket update

Another user changes it from the queue screen

Each place has to remember to update every badge. Miss one, and you get a sync bug. And these bugs are silent — no error, no crash, just wrong UI.

How React fixes this
In React, you don’t update each badge manually. You store one state and render all badges from it:

jsx
function DoctorView({ doctor }) {
  return (
    <>
      <ProfileCard>
        <Badge verified={doctor.isVerified} />
      </ProfileCard>

      <QueueList>
        <Badge verified={doctor.isVerified} />
      </QueueList>

      <AppointmentModal>
        <Badge verified={doctor.isVerified} />
      </AppointmentModal>
    </>
  );
}
To verify the doctor, you only change the state:

jsx
setDoctor({ ...doctor, isVerified: true });
React then re-renders all three badges because they all read from the same doctor.isVerified.

You can’t forget one — there’s only one source of truth.

The one-line takeaway
A state synchronization bug = you changed the data, but forgot to update every copy of it on the screen.
React prevents it by making the UI a reflection of state, not a set of manual edits you have to keep in sync.