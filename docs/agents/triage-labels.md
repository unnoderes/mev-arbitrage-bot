# Triage Label Vocabulary

This project uses the following five canonical triage labels. The `triage` skill applies these labels based on issue analysis and routing decisions.

## Label Mapping

| Role | Label String | Description |
|------|-------------|-------------|
| Needs Triage | `needs-triage` | New or reopened issue awaiting initial classification |
| Needs Info | `needs-info` | Blocked on clarification or missing details from reporter |
| Ready for Agent | `ready-for-agent` | Triaged and ready for AI agent implementation |
| Ready for Human | `ready-for-human` | Triaged and ready for human developer attention |
| Won't Fix | `wontfix` | Closed as out-of-scope, duplicate, or declined |

## Usage

When the `triage` skill processes issues, it:
1. Reads the issue description and context
2. Determines the appropriate triage state
3. Applies the corresponding label from this mapping
4. Moves the issue to the appropriate workflow state

Skills that create issues should apply `needs-triage` by default unless the classification is already clear.
