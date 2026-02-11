
## First step is to create this secret

kubectl create secret generic ops-manager-admin-secret \
  --from-literal=Username="admin@test.com" \
  --from-literal=Password="Admin@123" \
  --from-literal=FirstName="admin" \
  --from-literal=LastName="admin"
