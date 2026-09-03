# BuildExpress — UI Consistency Upgrade

A drop-in design system for `com.build.buildexpress`, built against your actual repo
(Java + XML, minSdk 24, Material Components 1.12, 81 layouts, 60+ activities).

**Guiding rule: every `android:id` in the redesigned layouts is byte-identical to the
original.** No `findViewById` breaks. No adapter rewrites. Zero Java changes required.

---

## Why your screens look inconsistent — measured, not guessed

I scanned all 81 layouts in `app/src/main/res/layout`:

| Problem | Count | Effect |
|---|---:|---|
| Hardcoded hex colours | **294** across **56** files | 39 different greys/oranges; nothing matches |
| Text smaller than 12sp | **70** instances (down to **7sp**) | Illegible; fails accessibility |
| Distinct card corner radii | **8** different values (8/12/16/20/24/30/45/100dp) | Every card a different shape |
| Legacy `androidx.cardview.CardView` | **41** files | No ripple, no stroke, no theming |
| Plain `<Button>` instead of `MaterialButton` | **40** files | Ignores theme, inconsistent height |

Fixing the theme layer fixes most of this globally, without touching 81 files by hand.

---

## What's in this package

```
res/values/colors.xml          REPLACE  — full semantic palette (legacy names kept)
res/values/dimens.xml          NEW      — 4dp spacing + type scale
res/values/styles.xml          REPLACE  — theme + all widget styles
res/values-night/colors.xml    NEW      — dark mode palette
res/values-night/styles.xml    NEW      — dark mode theme
res/color/*.xml                NEW  (6) — state selectors (pressed/checked/disabled)
res/drawable/*.xml             NEW (16) — pills, gradients, steppers, chat bubbles
res/anim/*.xml                 NEW  (6) — screen + list transitions
res/layout/item_row.xml        REPLACE  — product grid card
res/layout/cart_row.xml        REPLACE  — cart row
res/layout/item_paint_homerun.xml REPLACE — paint card
res/layout/view_empty_state.xml   NEW   — reusable empty state
res/layout/view_bottom_cta.xml    NEW   — sticky bottom action bar
```

Two files intentionally overwrite yours in place: `bg_header_gradient.xml` and
`rounded_search_bg.xml` — same filenames, restyled.

---

## Install

```bash
cd /path/to/my-android-app
cp -r /path/to/buildexpress-ui/res/* app/src/main/res/
```

Then **Build → Clean Project**, then **Rebuild**.

Nothing else is required — because `AppTheme` now sets `materialButtonStyle`,
`materialCardViewStyle`, `chipStyle`, `textInputStyle` and the shape system as
global defaults, **every Material widget in all 81 layouts restyles itself**.

---

## Phase 1 — theme only (5 minutes, zero risk)

Copy the files above and rebuild. Immediately, app-wide:

- One orange (`#FF7A00`) everywhere instead of three.
- Every button: 48dp tall, 14dp radius, sentence case, correct disabled + pressed states.
- Every card: 18dp radius, 1dp hairline stroke, ripple on tap.
- Every chip: pill-shaped with proper checked state.
- Every text field: outlined, 14dp radius, orange focus ring.
- White status bar with dark icons (replaces the orange bar — reads far more premium).
- Slide transitions between activities.

Your hardcoded hex colours still override the theme in individual layouts —
that's Phase 2.

---

## Phase 2 — kill the hardcoded colours (the big win)

`fix_hardcoded_colors.sh` (included) rewrites the 39 stray hex values across 56
layouts into theme references. It backs everything up first.

```bash
cd /path/to/my-android-app
bash /path/to/buildexpress-ui/fix_hardcoded_colors.sh
```

Review with `git diff`, then rebuild. If anything looks wrong:
`bash fix_hardcoded_colors.sh --restore`

---

## Phase 3 — screen-by-screen polish

Work in this order — highest user impact first:

### 1. `activity_item_list.xml` (Home — the most-seen screen)
- Wrap the product `RecyclerView` in `GridLayoutManager(this, 2)`.
  The new `item_row` is `match_parent`, so it fills each column properly
  (the old 110dp fixed width left dead space on large phones).
- Add to the RecyclerView so the last row clears the bottom nav:
  ```xml
  android:paddingBottom="@dimen/list_bottom_clearance"
  android:clipToPadding="false"
  ```
- Style section headers `tvBrowseCategories` / `tvBrowseMaterials`
  with `style="@style/Text.SectionTitle"`.
- Give `bottomNav` `style="@style/AppBottomNav"`.
- Animate the grid in Java:
  ```java
  recyclerView.setLayoutAnimation(
      AnimationUtils.loadLayoutAnimation(this, R.anim.layout_anim_fall_down));
  ```

### 2. `activity_cart.xml`
- Replace the checkout footer with `<include layout="@layout/view_bottom_cta" />`.
- Add the empty state:
  ```java
  View empty = findViewById(R.id.emptyState);
  ((TextView) empty.findViewById(R.id.emptyTitle)).setText("Your cart is empty");
  empty.setVisibility(cartItems.isEmpty() ? View.VISIBLE : View.GONE);
  ```

### 3. `activity_product_detail.xml` / `activity_item_detail.xml`
- Sticky `view_bottom_cta` with the "Add to cart" action.
- Image slider 1:1, `fitCenter`, `@color/bg_subtle` behind it.
- Price block: `@style/Text.Display` + `@style/Text.PriceStruck` + discount badge.

### 4. `activity_login.xml`, `activity_register.xml`, `activity_otp_verification.xml`
- All fields → `TextInputLayout` (inherits `@style/AppTextInput` automatically).
- Primary CTA `match_parent`, `@dimen/button_height`.
- Same `@dimen/screen_padding_h` gutter on all three so the logo doesn't jump
  between screens.

### 5. Chat screens (`support_chat`, `service_chat`, `quick_order_chat`)
- `item_chat_sent` → `android:background="@drawable/bg_chat_sent"`
- `item_chat_received` → `android:background="@drawable/bg_chat_received"`
- Screen background → `@color/chat_screen_bg`
- Max bubble width ~75% so long messages stay readable.

### 6. `activity_my_orders.xml` + `order_row.xml`
Use the status pill styles instead of ad-hoc colours:
`@style/Badge.Warning` pending · `@style/Badge.Info` confirmed ·
`@style/Badge.Success` delivered · red `Badge.Tag` cancelled.

### 7. Admin screens
Lower priority — they're internal. The Phase 1 theme already normalises them.

---

## Cheat sheet

| Instead of | Use |
|---|---|
| `android:textSize="10sp"` | `@dimen/text_caption` (12sp minimum) |
| `android:textColor="#333"` | `@color/text_primary` |
| `android:textColor="#666"` / `#999` | `@color/text_secondary` / `@color/text_tertiary` |
| `android:background="#FFF"` | `@color/bg_card` |
| `#FF9800` | `@color/brand_orange` |
| `#2E7D32` | `@color/success_green` |
| `#F44336` | `@color/error_red` |
| `padding="10dp"` | `@dimen/space_md` |
| `cardCornerRadius="12dp"` | `@dimen/radius_lg` |
| `<Button>` | `<com.google.android.material.button.MaterialButton>` |
| `androidx.cardview.widget.CardView` | `com.google.android.material.card.MaterialCardView` |
| section header TextView | `style="@style/Text.SectionTitle"` |

---

## Dark mode

`values-night/` is included but **do not ship it until Phase 2 is done** — hardcoded
`#FFFFFF` backgrounds will produce white-on-white. Until then, pin light mode in
`SplashActivity.onCreate` before `super`:

```java
AppCompatDelegate.setDefaultNightMode(AppCompatDelegate.MODE_NIGHT_NO);
```

Remove that line when you're ready to enable dark mode.

---

## Two things to fix while you're in here

1. **`MainActivity` is a stub** — `setContentView(tv)` on a bare `TextView`, yet it's
   registered in the Manifest. Either delete it or make it a real screen; right now
   anything that routes there shows a blank white screen.

2. **`local.properties` and `firestore.rules` are in your public repo.** Check that
   your Firestore rules aren't `allow read, write: if true`. That matters more than
   any of this UI work.

---
---

# v2 UPDATE — HomeRun-matched UI + your app icon

Based on the HomeRun screenshots you sent and your BuildExpress logo.

## Design decision: orange, not green

HomeRun uses **green CTAs and a yellow checkout**. You said orange is your brand
colour, so those CTAs are **orange** here. What I *did* keep from HomeRun are the
elements that carry *meaning* rather than brand identity:

| Element | Colour | Why kept |
|---|---|---|
| Announcement strip | amber `#FFC629` | store-hours notice must stand apart from brand |
| Benefits bar | maroon `#8C1D18` | trust band; orange-on-orange would vanish |
| Cashback banner | dark green `#0B3D18` | money/savings reads green universally |
| "60 Mins" badge | green `#0F8A43` | speed/go signal |
| Discount badge | amber `#FFC629` | maximum contrast on product photos |
| Checkout CTA | amber pill | deliberately distinct from "Add to cart" orange |

Everything else — Add buttons, steppers, focus rings, chips, chat bubbles, links —
is your orange `#F26100`, sampled from your logo's gradient.

## App icon

Generated from the logo you supplied, at every density:

```
res/mipmap-{m,h,xh,xxh,xxxh}dpi/ic_launcher.png            squircle
res/mipmap-{m,h,xh,xxh,xxxh}dpi/ic_launcher_round.png      circular
res/mipmap-{m,h,xh,xxh,xxxh}dpi/ic_launcher_foreground.png adaptive, 66dp safe zone
res/mipmap-anydpi-v26/ic_launcher.xml                      adaptive icon
res/mipmap-anydpi-v26/ic_launcher_round.xml                adaptive round
res/drawable/ic_launcher_background.xml                    orange gradient
res/drawable-*/ic_stat_notify.png                          white notification silhouette
res/drawable-*/logo_buildexpress.png                       splash/header logo
play_store_icon_512.png                                    Play Store listing
```

The adaptive icon includes a `<monochrome>` layer, so it themes correctly on
Android 13+. Your Manifest already points at `@mipmap/ic_launcher` and
`@mipmap/ic_launcher_round` — **no Manifest change needed**.

Use the notification icon in `OrderNotificationHelper` / `AdminNotificationService`:

```java
.setSmallIcon(R.drawable.ic_stat_notify)
.setColor(ContextCompat.getColor(this, R.color.brand_orange))
```

Android requires notification icons be a **white silhouette on transparent** —
your current coloured launcher icon would render as a grey blob.

## New HomeRun components

| File | Purpose |
|---|---|
| `view_announcement_strip.xml` | amber "Open 8 am to 8 pm" marquee |
| `view_benefits_bar.xml` | maroon Pay on Delivery / Cashback / Free Delivery |
| `view_location_header.xml` | logo · 60-min badge · pincode · cart badge · search |
| `view_cashback_banner.xml` | dark-green progress banner for the cart |
| `dialog_variant_picker.xml` | bottom sheet w/ size chips + per-row Add |
| `item_variant_row.xml` | swatch · name · struck MRP · price · Add |

### Home screen assembly order

```xml
<include layout="@layout/view_announcement_strip" .../>   <!-- amber strip -->
<include layout="@layout/view_location_header"   .../>   <!-- header+search -->
<include layout="@layout/view_benefits_bar"      .../>   <!-- maroon band -->
<!-- banner slider, categories, product grid ... -->
```

### Product card additions

`item_row.xml` keeps all 19 original IDs and adds 6 optional ones. They default to
sensible values; set them in `ItemAdapter` only if you have the data:

```java
holder.tvDiscountPct.setText("29% OFF");
holder.tvDiscountPct.setVisibility(discount > 0 ? View.VISIBLE : View.GONE);
holder.tvMrp.setPaintFlags(holder.tvMrp.getPaintFlags() | Paint.STRIKE_THRU_TEXT_FLAG);
holder.tvCashback.setVisibility(View.VISIBLE);
holder.tvRowBulk.setText("Unlock Bulk Prices of ₹325");
```

New optional IDs: `tvDiscountPct`, `imgBrandLogo`, `tvDeliveryMeta`,
`tvFreeDelivery`, `tvMrp`, `tvCashback`.

## ID safety — re-verified for v2

I checked the replaced layouts against the Java that actually uses them:

| Layout | IDs preserved | Used by |
|---|---|---|
| `item_row.xml` | 19/19 (+6 new) | `ItemAdapter` |
| `cart_row.xml` | 14/14 | `CartAdapter` |
| `item_paint_homerun.xml` | 12/12 | paint adapter |
| `dialog_variant_picker.xml` | 4/4 (+3 new) | `ItemAdapter`, `ItemListActivity`, `ItemDetailActivity`, `CategoryGalleryAdapter` |
| `item_variant_row.xml` | 5/5 | same four files |

`btnClosePopup` is still a Button subclass, so
`findViewById(R.id.btnClosePopup).setOnClickListener(...)` keeps working.

## Install

```bash
cd /path/to/my-android-app
cp -r /path/to/buildexpress-ui/res/* app/src/main/res/
bash /path/to/buildexpress-ui/fix_hardcoded_colors.sh
```

Then **Build → Clean Project → Rebuild**. Uninstall and reinstall the app to force
the launcher to refresh its icon cache.
