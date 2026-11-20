# ISE3230
import gurobipy as gp
from gurobipy import GRB

m = gp.Model("hiring_schedule")
m.Params.LogToConsole = 0

# variables to read in: for each candidate: wage, hiring_cost, min | for each shift: hours, required_employees - xinxin
candidate = range(1, n_candidates + 1)
shift = range(1, n_shifts + 1)

y = m.addVars(candidate, lb=0, vtype=GRB.BINARY, name="hiring")
x = m.addVars(candidate, shift, vtype=GRB.BINARY, name="scheduling")

m.setObjective(gp.quicksum(x[i, s]* wage[i] * hours[s] for i in candidate for s in shift) + gp.quicksum(y[i]*hiring_cost[i] for i in candidate), GRB.MINIMIZE)

# Add constraints for hiring and scheduling (cannont schedule if a candidate is not hiried)
for i in candidate:
  for s in shift:
    m.addConstr(x[i,s] <= y[i])

# Add constraints for min working hours
for i in candidate:
  m.addConstr(gp.quicksum(x[i,s] * hours[s] for s in shift) >= min[i] * y[i])

# Add constraints for max working hours
for i in candidate:
  m.addConstr(gp.quicksum(x[i,s] * hours[s] for s in shift) <= max[i] * y[i])

