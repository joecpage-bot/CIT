# CIT Coach Sign-Up — Handoff (2026-10-19)

Status and instructions for whoever picks this up next (person or AI assistant).

## Goal
Make it easy for CIT coaches (many outside the City of Golden tenant) to sign up for, change, and see their coaching shifts. SharePoint stays the source of truth. Bookings was rejected.

## Systems (City of Golden, Microsoft 365 GCC)
- **SharePoint site:** https://cityofgolden.sharepoint.com/sites/jeffcocit
- **List:** "Coach Sign-up" (URL path `/Lists/Coach Signup`, GUID `3fee83f2-4d02-4cec-8a24-293403ce1e98`)
  - Key internal fields: `Title`, `StartDate`, `EndDate`, `BookingID`, `BookingStatus` (Scheduled / Waitlist / Cancelled), `BookingChangeType`, `BookingDetails`, `BookingLocation`, `Coach_x0020_Email`, `First_Name`, `Last_Name`, `Notes`
  - BookingID format: `email-COACH|SHADOW-YYYYMMDD` (lowercase email)
- **Power Automate (GCC):** https://make.gov.powerautomate.us, Default environment `Default-1b4e6cc8-7e6d-42c7-ba21-115d29acf480`
  - "CIT Coach Sign-up - Forms Capacity Sync" — flow ID `15da6dc3-1cd1-401e-9658-4cbc072d709b`
  - "CIT Coach Cancellation / Change Request - Forms Sync" — flow ID `208725dc-f3cb-4690-8d95-a571a519e1af`
- **Forms (owner Joe Page):**
  - "2026 Basic CIT Coach Sign-Up" — form ID `yGxOG21-x0K6IRFdKaz0gHmLto_r7xxAmqY7LozvDLJUNkRJV1VLN1FTNzlKSk1RSThEN01TNVAwNi4u`
  - "CIT Coach Cancellation / Change Request" — https://forms.cloud.microsoft/g/CUSL38UALh

## Done on 2026-10-19
1. **Data cleanup in Coach Sign-up list** (all verified via REST):
   - Drea Jones' October rows (IDs 220, 221) retitled from "October 22/23" to October 29/30 to match their Start dates; unique BookingIDs set. She was emailed separately.
   - All July rows' location set to McNeil Hall; all October rows already Arvada Fire.
   - Old non-unique `...-JUL21` BookingIDs rewritten to the new format (18 rows). No duplicate BookingIDs remain.
   - Angela Gosalvez July 23 (ID 212): kept Scheduled; notes/change type now say she was moved off the waitlist on 2026-07-07.
2. **Capacity check reviewed:** each shift block counts rows matching exact `StartDate` + exact `Title`, excluding Cancelled/Waitlist, and schedules if count < 9. Logic is correct; earlier July waitlists came from an older flow version (current flow created 2026-07-14). Title mismatches were the real counting risk.
3. **"Full schedule" email added to both flows** (saved, Flow checker clean, tested end to end):
   - Sign-up flow: before "Send an email (V2)", added **Get coach schedule** (SharePoint Get items; filter `Coach_x0020_Email eq '<formEmail>' and StartDate ge '<utcNow>' and BookingStatus ne 'Cancelled'`, order `StartDate asc`) and **Build schedule table** (Create HTML table, custom columns Shift + Status). Email body ends with "Your full upcoming CIT schedule…" + table output.
   - Cancellation flow: same two steps inside True → For each, between "Update item" and "Send coach confirmation email". Email address comes from form field `r64602f342b564569810f2c1c9c4661e7`.
   - Shift column expression: `concat(formatDateTime(convertFromUtc(item()?['StartDate'],'Mountain Standard Time'),'ddd, MMM d'),' | ',formatDateTime(convertFromUtc(item()?['StartDate'],'Mountain Standard Time'),'h:mm tt'),' - ',formatDateTime(convertFromUtc(item()?['EndDate'],'Mountain Standard Time'),'h:mm tt'),' | ',if(contains(item()?['Title'],'Shadow'),'New Coach / Shadow','Coach'))`
   - Test sign-up + cancellation with jpage@cityofgolden.gov both produced correct emails. Test row (ID 274) moved to the SharePoint recycle bin. Two test responses remain in the Forms results (safe to delete).

## Still open (in priority order)
1. **Empty-schedule message (small):** when a coach has no remaining shifts, the table shows only headers. Add a Condition on `empty(body('Get_coach_schedule')?['value'])` and show "You have no upcoming CIT shifts." instead.
2. **Rebuild the sign-up flow before the next class (bigger):** the flow has 16 copy-pasted blocks with dates and titles hard-coded (October blocks are still *named* "October 20–23" though they check Oct 27–30). Replace with a "CIT Shifts" list (Shift key, Role, Start, End, Location, Capacity, Form choice text) and one Apply to each over the form's selected choices. Count by StartDate + Role instead of exact Title.
3. **Cancellation matching:** cancel form asks only for a date, so a coach with both a Coach and a Shadow shift on the same day is ambiguous. Consider using the same shift choice list as the sign-up form.
4. Optional: add a Location column to the schedule table: `first(split(coalesce(item()?['BookingLocation'],''),decodeUriComponent('%0A')))`.

## Gotchas for whoever continues
- Tenant is **GCC**: use make.gov.powerautomate.us, not make.powerautomate.com.
- In the new Power Automate designer, the expression editor flags `item()` as "has a problem" inside Create HTML table; clicking Add a second time saves it, and it runs fine.
- Rich-text email bodies that are a single `concat(...)` expression: click into the body, Ctrl+End, then type/insert after the token.
- Shifts run 2:30–5:30 PM Tue–Thu and 12:30–3:30 PM Fri, Mountain time; 9 Coach + 9 Shadow seats per block.
