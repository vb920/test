``` python
"""Reference repair for the Workflow practice adapter."""

from copy import deepcopy


TEAMS = {
    "team-engineering": {
        "_id": "team-engineering",
        "name": "Engineering",
        "key": "ENG",
        "icon": "⚙",
    },
    "team-design": {
        "_id": "team-design",
        "name": "Design",
        "key": "DES",
        "icon": "✦",
    },
}


def _seed_issue(identifier, title, team_id):
    team = TEAMS[team_id]
    return {
        "_id": f"issue-{identifier.lower()}",
        "identifier": identifier,
        "title": title,
        "description": "",
        "status": "todo",
        "priority": "no_priority",
        "team": deepcopy(team),
        "assignee": None,
        "creator": "user-alex",
        "parentIssue": None,
        "labels": [],
        "createdAt": "2024-01-01T00:00:00.000Z",
        "updatedAt": "2024-01-01T00:00:00.000Z",
    }


class WorkflowStore:
    def reset(self):
        self.issues = [
            _seed_issue("ENG-1", "Create admin dashboard", "team-engineering"),
            _seed_issue("ENG-2", "Implement data export feature", "team-engineering"),
            _seed_issue("DES-1", "Refresh component library", "team-design"),
        ]
        self.activities = []

    def list_issues(self, team_id):
        self.require_team(team_id)
        return deepcopy(
            [
                issue
                for issue in self.issues
                if issue["team"]["_id"] == team_id and issue["parentIssue"] is None
            ]
        )

    def list_sub_issues(self, parent_identifier):
        parent = self.require_issue(parent_identifier)
        return deepcopy(
            [
                issue
                for issue in self.issues
                if issue["parentIssue"] == parent["_id"]
            ]
        )

    def create_issue(self, payload):
        team = self.require_team(payload["teamId"])
        parent_id = payload.get("parentIssue")
        if parent_id is not None:
            parent = self.require_issue_by_id(parent_id)
            if parent["team"]["_id"] != team["_id"]:
                raise KeyError("Issue not found")

        prefix = f"{team['key']}-"
        numbers = [
            int(issue["identifier"].removeprefix(prefix))
            for issue in self.issues
            if issue["identifier"].startswith(prefix)
        ]
        identifier = f"{team['key']}-{max(numbers, default=0) + 1}"
        issue = {
            "_id": f"issue-{identifier.lower()}",
            "identifier": identifier,
            "title": payload["title"],
            "description": payload.get("description", ""),
            "status": payload.get("status") or "todo",
            "priority": payload.get("priority") or "no_priority",
            "team": deepcopy(team),
            "assignee": payload.get("assignee"),
            "creator": "user-alex",
            "parentIssue": parent_id,
            "labels": list(payload.get("labels", [])),
            "createdAt": "2024-01-01T00:00:00.000Z",
            "updatedAt": "2024-01-01T00:00:00.000Z",
        }
        self.issues.append(issue)
        self.activities.append(
            {
                "action": "issue.created",
                "issueId": issue["_id"],
                "identifier": identifier,
            }
        )
        return deepcopy(issue)

    def require_team(self, team_id):
        if team_id not in TEAMS:
            raise KeyError("Team not found")
        return TEAMS[team_id]

    def require_issue(self, identifier):
        for issue in self.issues:
            if issue["identifier"] == identifier:
                return issue
        raise KeyError("Issue not found")

    def require_issue_by_id(self, issue_id):
        for issue in self.issues:
            if issue["_id"] == issue_id:
                return issue
        raise KeyError("Issue not found")


store = WorkflowStore()
store.reset()
```
