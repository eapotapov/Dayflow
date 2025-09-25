# Categories System Documentation

## Overview

The categories system in Dayflow allows users to organize their activities into meaningful groups. Categories are assigned by AI during analysis and can be customized by users to match their workflow.

## Default Categories

### System Categories (Cannot be deleted)

#### Work (Purple - #B984FF)
**Definition**: "Professional, school, or career-focused tasks (coding, design, meetings, research)"
**Examples**:
- Writing code in IDE
- Attending Zoom meetings
- Research for work projects
- Email correspondence
- Document preparation

#### Personal (Blue - #6AADFF)
**Definition**: "Intentional non-work activities for life, wellbeing, hobbies, or personal errands"
**Examples**:
- Online shopping
- Trip planning
- Personal finance
- Learning new skills
- Health appointments

#### Distraction (Red - #FF5950)
**Definition**: "Unplanned, aimless, or compulsive time sinks (social media, doomscrolling, non-work videos, rabbit holes)"
**Examples**:
- Social media scrolling
- YouTube rabbit holes
- Reddit browsing
- News doomscrolling
- Random web surfing

#### Idle (Gray - #A0AEC0)
**Definition**: "Mark sessions where the user is idle for most of the time"
**Special Rules**:
- System category (isSystem: true)
- Cannot be deleted (isIdle: true)
- Only assigned when >50% of time period shows no activity
- Auto-detected by AI when screen is static for 5+ minutes

## Data Model

### TimelineCategory Structure
```swift
struct TimelineCategory {
    var id: UUID                // Unique identifier
    var name: String           // Display name
    var colorHex: String       // Hex color code
    var details: String        // Description for AI
    var order: Int            // Display order
    var isSystem: Bool        // System category flag
    var isIdle: Bool          // Idle category flag
    var isNew: Bool           // Newly created flag
    var createdAt: Date       // Creation timestamp
    var updatedAt: Date       // Last update timestamp
}
```

### LLMCategoryDescriptor
Simplified structure sent to AI:
```swift
struct LLMCategoryDescriptor {
    let id: UUID
    let name: String
    let description: String?
    let isSystem: Bool
    let isIdle: Bool
}
```

## Category Store

### Singleton Instance
```swift
@MainActor
final class CategoryStore: ObservableObject {
    @Published private(set) var categories: [TimelineCategory] = []
}
```

### Key Methods

#### Adding Categories
```swift
func addCategory(name: String, colorHex: String? = nil) {
    // Creates user-defined category
    // Auto-assigns next order number
    // Sets isSystem: false
}
```

#### Updating Categories
```swift
func updateCategory(id: UUID, mutate: (inout TimelineCategory) -> Void)
func renameCategory(id: UUID, to: String)
func assignColor(_ hex: String, to: UUID)
func updateDetails(_ details: String, for: UUID)
```

#### Managing Categories
```swift
func removeCategory(id: UUID)  // Only non-system categories
func reorderCategories(_ idsInOrder: [UUID])
```

## Storage

### Persistence
Categories are stored in UserDefaults as JSON:
```swift
UserDefaults.standard.set(data, forKey: "colorCategories")
```

### Migration Support
Handles legacy format from older versions:
```swift
struct LegacyColorCategory: Codable {
    let id: Int64
    var name: String
    var color: String?
    var details: String
    var isNew: Bool?
}
```

## AI Integration

### How AI Assigns Categories

1. **Category List Sent to LLM**
```swift
USER CATEGORIES (choose exactly one label):
1. "Work" — Professional, school, or career-focused tasks
2. "Personal" — Intentional non-work activities
3. "Distraction" — Unplanned, aimless time sinks
4. "Idle" — User is idle for most of this period
```

2. **AI Analysis Process**
```
Screen Content → Activity Purpose → Match to Category → Apply to Card
```

3. **Category Normalization**
After AI responds, normalize the category:
```swift
func normalizeCategory(_ raw: String) -> String {
    // Case-insensitive matching
    // Handle variations ("idle time" → "Idle")
    // Fallback to first category if no match
}
```

### Prompt Integration
Categories are dynamically inserted into prompts:
```swift
private func categoriesSection(from descriptors: [LLMCategoryDescriptor]) -> String {
    // Build formatted category list
    // Include descriptions
    // Add special rules for idle
}
```

## User Customization

### Adding Custom Categories
Users can create categories for specific needs:
- "Learning" - Online courses and tutorials
- "Side Project" - Personal coding projects
- "Entertainment" - Intentional media consumption
- "Communication" - Calls, messages, emails

### Customization Options
- **Name**: Any string (trimmed, non-empty)
- **Color**: Hex color code for timeline display
- **Details**: Description to guide AI assignment
- **Order**: Drag to reorder in UI

### Best Practices for Custom Categories
1. **Be Specific**: "Video Editing" vs "Creative Work"
2. **Add Descriptions**: Help AI understand when to apply
3. **Avoid Overlap**: Keep categories distinct
4. **Use Colors Wisely**: High contrast for easy scanning

## UI Integration

### Category Picker
```swift
TimelineCardColorPicker(
    selectedCategory: $selectedCategory,
    categories: categoryStore.categories
)
```

### Timeline Filtering
```swift
// Filter cards by category
let filtered = cards.filter { $0.category == selectedCategory }
```

### Settings Management
Users can manage categories in Settings:
- Add new categories
- Edit existing ones
- Reorder list
- Delete custom categories

## Category Assignment Logic

### Decision Flow
```
1. Check idle percentage
   ├─ >50% idle → Assign "Idle"
   └─ <50% idle → Continue

2. Analyze activity purpose
   ├─ Professional/School → "Work"
   ├─ Intentional personal → "Personal"
   ├─ Aimless browsing → "Distraction"
   └─ Custom match → User category

3. Apply normalization
   └─ Ensure exact match with defined categories
```

### Edge Cases
- **Multi-purpose activities**: AI picks dominant purpose
- **Unknown activities**: Falls back to best guess
- **Transition periods**: Included in larger activity
- **Brief interruptions**: Stay with main category

## Database Integration

### Timeline Cards Table
```sql
CREATE TABLE timeline_cards (
    category TEXT NOT NULL,      -- Category name
    subcategory TEXT,            -- Optional subcategory
    ...
)
```

### Querying by Category
```sql
SELECT * FROM timeline_cards
WHERE category = 'Work'
  AND day = '2025-01-15'
ORDER BY start_ts ASC
```

## Analytics

### Category Usage Tracking
```swift
AnalyticsService.capture("category_assigned", [
    "category": categoryName,
    "duration_minutes": duration,
    "ai_confidence": confidence
])
```

### Category Distribution
```swift
func calculateCategoryBreakdown(for date: Date) -> [String: TimeInterval] {
    // Sum time per category
    // Return percentage breakdown
}
```

## Migration and Compatibility

### Ensuring Idle Category
Always present after migration:
```swift
static func ensureIdleCategoryPresent(in categories: [TimelineCategory]) -> [TimelineCategory] {
    if !categories.contains(where: { $0.isIdle }) {
        // Add idle category
    }
}
```

### Default Categories Reset
Users can reset to defaults:
```swift
categories = CategoryPersistence.defaultCategories
```

## Future Enhancements

### Planned Features
- **Sub-categories**: Hierarchical organization
- **Auto-suggest**: AI-recommended categories
- **Templates**: Pre-built category sets
- **Import/Export**: Share configurations

### API Extensions
```swift
// Future sub-category support
struct SubCategory {
    let parent: UUID
    let name: String
    let rules: [String]
}

// Future auto-detection rules
struct CategoryRule {
    let apps: [String]
    let websites: [String]
    let keywords: [String]
}
```

## Troubleshooting

### Common Issues

#### Wrong Category Assigned
**Solution**: Update category description to be more specific

#### Category Not Available
**Solution**: Ensure category is saved before analysis runs

#### Idle Over-detected
**Solution**: Adjust idle detection threshold in prompts

#### Custom Category Ignored
**Solution**: Check description clarity, avoid overlaps with defaults

### Debug Commands
```swift
// View all categories
print(CategoryStore.descriptorsForLLM())

// Test category matching
let normalized = normalizeCategory("work time")

// Export category configuration
let json = try! JSONEncoder().encode(categories)
```