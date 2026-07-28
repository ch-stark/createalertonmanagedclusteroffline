echo "testcase: blocking communication from spoke to hub"

DURATION_MIN=20
POLICY_NAME="block-hub-egress"
NAMESPACES=("open-cluster-management-agent" "open-cluster-management-agent-addon")

for ns in "${NAMESPACES[@]}"; do
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ${POLICY_NAME}
  namespace: ${ns}
spec:
  podSelector: {}
  policyTypes:
  - Egress
EOF
done

echo "Egress blocked. Restoring in ${DURATION_MIN} min... (Ctrl-C to restore early)"
sleep $(( DURATION_MIN * 60 ))

for ns in "${NAMESPACES[@]}"; do
  kubectl delete networkpolicy "${POLICY_NAME}" -n "${ns}" --ignore-not-found
done
echo "Egress restored."
networkpolicy.networking.k8s.io/block-hub-egress created
