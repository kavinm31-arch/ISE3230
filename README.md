# ISE3230
import gurobipy as gp
from gurobipy import GRB

m = gp.Model("hiring_schedule")
m.Params.LogToConsole = 0

y = m.addVars(18, lb=0, vtype=GRB.INTEGER, name="y")

m.setObjective(, GRB.MINIMIZE)
