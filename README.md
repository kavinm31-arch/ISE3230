# ISE3230
import gurobipy as gp
from gurobipy import GRB

m = gp.Model("hiring_schedule")
m.Params.LogToConsole = 0

from pathlib import Path
import pandas as pd

# Locate the Excel file beside this Python script.
excel_path = (
    Path(__file__).resolve().parent
    / "Group18_Existing_Workbook_Single_Optimization.xlsx"
)

employees = pd.read_excel(
    excel_path, sheet_name="Employees", header=2
)
shifts = pd.read_excel(
    excel_path, sheet_name="Shifts", header=2
)
availability_table = pd.read_excel(
    excel_path, sheet_name="Availability", header=2
)

# Use the original Excel IDs consistently.
employees = employees.set_index("Employee ID")
shifts = shifts.set_index("Shift ID")
availability_table = availability_table.set_index("Employee ID")

candidate = employees.index.tolist()
shift = shifts.index.tolist()

# Employee parameters.
wage = employees["Hourly wage (USD)"].to_dict()
hiring_cost = employees["Weekly hiring cost (USD)"].to_dict()
min_hours = employees["Min hours if hired"].to_dict()
max_hours = employees["Max weekly hours"].to_dict()

# Shift parameters.
hours = shifts["Duration (hours)"].to_dict()
required_employees = shifts["Required employees"].to_dict()

# Check that all IDs are unique and the availability matrix aligns.
assert employees.index.is_unique, "Duplicate employee IDs"
assert shifts.index.is_unique, "Duplicate shift IDs"
assert availability_table.index.is_unique, "Duplicate availability IDs"
assert set(availability_table.index) == set(candidate)
assert set(availability_table.columns) == set(shift)

# Align matrix rows and columns with the model IDs.
availability_table = availability_table.loc[candidate, shift]

assert availability_table.isin([0, 1]).all().all(), (
    "Availability must contain only 0 or 1"
)

availability = {
    (i, s): int(availability_table.loc[i, s])
    for i in candidate
    for s in shift
}

# Group shift IDs by day for daily assignment constraints.
day_shifts = {
    day: group.index.tolist()
    for day, group in shifts.groupby("Day index")
}

# Check required numeric inputs.
employee_columns = [
    "Hourly wage (USD)",
    "Weekly hiring cost (USD)",
    "Min hours if hired",
    "Max weekly hours",
]
shift_columns = ["Duration (hours)", "Required employees"]

for column in employee_columns:
    assert pd.api.types.is_numeric_dtype(employees[column]), (
        f"Non-numeric employee input: {column}"
    )
    assert employees[column].notna().all(), (
        f"Missing employee input: {column}"
    )
    assert (employees[column] >= 0).all()

for column in shift_columns:
    assert pd.api.types.is_numeric_dtype(shifts[column]), (
        f"Non-numeric shift input: {column}"
    )
    assert shifts[column].notna().all(), (
        f"Missing shift input: {column}"
    )

for i in candidate:
    assert min_hours[i] <= max_hours[i], (
        f"Minimum hours exceed maximum hours for {i}"
    )

for s in shift:
    assert hours[s] > 0
    assert required_employees[s] >= 0
    assert required_employees[s] == int(required_employees[s])
    assert sum(availability[i, s] for i in candidate) >= (
        required_employees[s]
    ), f"Not enough available employees for {s}"

print(f"Loaded {n_candidates} candidates and {n_shifts} shifts.")

y = m.addVars(candidate, lb=0, vtype=GRB.BINARY, name="hiring")
x = m.addVars(candidate, shift, vtype=GRB.BINARY, name="scheduling")

m.setObjective(gp.quicksum(x[i, s]* wage[i] * hours[s] for i in candidate for s in shift) + gp.quicksum(y[i]*hiring_cost[i] for i in candidate), GRB.MINIMIZE)

# Add constraints for hiring and scheduling (cannont schedule if a candidate is not hiried)
for i in candidate:
  for s in shift:
    m.addConstr(x[i,s] <= y[i])

# Add constraints for min working hours
for i in candidate:
  m.addConstr(gp.quicksum(x[i,s] * hours[s] for s in shift) >= min_hours[i] * y[i])

# Add constraints for max working hours
for i in candidate:
  m.addConstr(gp.quicksum(x[i,s] * hours[s] for s in shift) <= max_hours[i] * y[i])

# Add constraints for staffing requirments (Kavin)
for s in shift:
    m.addConstr(gp.quicksum(x[i,s] for i in candidate) >= required_employees[s])

# Add constraints for employee availability (Kavin)
for i in candidate:
    for s in shift:
        m.addConstr(x[i,s] <= availability[i,s])

# Add constraints for daily shift limits (Kavin)
for i in candidate:
    for d in day_shifts:
        m.addConstr(gp.quicksum(x[i,s] for s in day_shifts[d]) <= 1)


# Optimize model (Kavin)
m.optimize()
if m.Status == GRB.OPTIMAL:
    print(f"Total weekly cost: ${m.ObjVal:,.2f}")
    for i in candidate:
        if y[i].X > 0.5:
            assigned = [s for s in shift if x[i,s].X > 0.5]
            total_hours = sum(hours[s] for s in assigned)
            print(f"{i}: {total_hours} hours, shifts {assigned}")
else:
    print(f"No optimal solution found (status {m.Status})")


