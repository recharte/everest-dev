# Vars
namespace = os.getenv('EVEREST_NAMESPACE', 'percona-everest')
print('Using namespace: %s' % namespace)
pxc_operator_version = os.getenv('PXC_OPERATOR_VERSION', '1.12.0')
print('Using PXC operator version: %s' % pxc_operator_version)
psmdb_operator_version = os.getenv('PSMDB_OPERATOR_VERSION', '1.15.0')
print('Using PSMDB operator version: %s' % psmdb_operator_version)
pg_operator_version = os.getenv('PG_OPERATOR_VERSION', '2.2.0')
print('Using PG operator version: %s' % pg_operator_version)

# Check for required env vars
backend_dir = os.getenv('EVEREST_BACKEND_DIR')
if not backend_dir:
  fail('EVEREST_BACKEND_DIR must be set')
backend_dir = os.path.abspath(backend_dir)
if not os.path.exists(backend_dir):
  fail('Backend dir does not exist: %s' % backend_dir)
print('Using backend dir: %s' % backend_dir)

frontend_dir = os.getenv('EVEREST_FRONTEND_DIR')
if not frontend_dir:
  fail('EVEREST_FRONTEND_DIR must be set')
frontend_dir = os.path.abspath(frontend_dir)
if not os.path.exists(frontend_dir):
  fail('Frontend dir does not exist: %s' % frontend_dir)
print('Using frontend dir: %s' % frontend_dir)

operator_dir = os.getenv('EVEREST_OPERATOR_DIR')
if not operator_dir:
  fail('EVEREST_OPERATOR_DIR must be set')
operator_dir = os.path.abspath(operator_dir)
if not os.path.exists(operator_dir):
  fail('Operator dir does not exist: %s' % operator_dir)
print('Using operator dir: %s' % operator_dir)

# Ensure operator's kustomize is installed
local('[ -x %s/bin/kustomize ] || make -C %s kustomize' % (operator_dir, operator_dir), quiet=True)

# Ensure frontend repo is initialized
local('make -C %s init' % (frontend_dir), quiet=True)

# Create namespace
load('ext://namespace', 'namespace_create', 'namespace_inject')
namespace_create(namespace)
k8s_resource(
  objects=[
	'%s:namespace' % namespace,
  ],
  new_name='namespace',
)


#################################
## Install DB engine operators ##
#################################

# PXC
pxc_operator_bundle_yaml = local(['curl', '-s', 'https://raw.githubusercontent.com/percona/percona-xtradb-cluster-operator/v%s/deploy/bundle.yaml' % pxc_operator_version], quiet=True)
k8s_yaml(namespace_inject(pxc_operator_bundle_yaml, namespace))
k8s_resource(
  workload='percona-xtradb-cluster-operator',
  objects=[
	'perconaxtradbclusterbackups.pxc.percona.com:customresourcedefinition',
	'perconaxtradbclusterrestores.pxc.percona.com:customresourcedefinition',
	'perconaxtradbclusters.pxc.percona.com:customresourcedefinition',
	'percona-xtradb-cluster-operator:serviceaccount',
	'percona-xtradb-cluster-operator:role',
	'service-account-percona-xtradb-cluster-operator:rolebinding',
  ],
  resource_deps = [
  	'namespace',
  ],
  new_name='pxc-operator',
  labels=["dbengines"]
)

# PSMDB
psmdb_operator_bundle_yaml = local(['curl', '-s', 'https://raw.githubusercontent.com/percona/percona-server-mongodb-operator/v%s/deploy/bundle.yaml' % psmdb_operator_version], quiet=True)
k8s_yaml(namespace_inject(psmdb_operator_bundle_yaml, namespace))
k8s_resource(
  workload='percona-server-mongodb-operator',
  objects=[
	'perconaservermongodbbackups.psmdb.percona.com:customresourcedefinition',
	'perconaservermongodbrestores.psmdb.percona.com:customresourcedefinition',
	'perconaservermongodbs.psmdb.percona.com:customresourcedefinition',
	'percona-server-mongodb-operator:serviceaccount',
	'percona-server-mongodb-operator:role',
	'service-account-percona-server-mongodb-operator:rolebinding',
  ],
  resource_deps = [
  	'namespace',
  ],
  new_name='psmdb-operator',
  labels=["dbengines"]
)

# PG
pg_operator_bundle_yaml = local(['curl', '-s', 'https://raw.githubusercontent.com/percona/percona-postgresql-operator/v%s/deploy/bundle.yaml' % pg_operator_version], quiet=True)
k8s_yaml(namespace_inject(pg_operator_bundle_yaml, namespace))
k8s_resource(
  workload='percona-postgresql-operator',
  objects=[
	'perconapgbackups.pgv2.percona.com:customresourcedefinition',
	'perconapgclusters.pgv2.percona.com:customresourcedefinition',
	'perconapgrestores.pgv2.percona.com:customresourcedefinition',
	'postgresclusters.postgres-operator.crunchydata.com:customresourcedefinition',
	'percona-postgresql-operator:serviceaccount',
	'percona-postgresql-operator:role',
	'service-account-percona-postgresql-operator:rolebinding',
  ],
  resource_deps = [
  	'namespace',
  ],
  new_name='pg-operator',
  labels=["dbengines"]
)

################################
####### Everest operator #######
################################

# Generate the Everest operator manifests
local_resource(
  'operator-gen-manifests',
  'make -C %s fmt manifests generate' % operator_dir,
  deps=['%s/api' % operator_dir],
  ignore=['%s/api/*/zz_generated.deepcopy.go' % operator_dir],
  labels=["everest-operator"]
)

# Build the Everest operator manager locally to take advantage of the go cache
local_resource(
  'operator-build',
  'make -C %s fmt; CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -C %s -o ./bin/manager ./main.go' % (operator_dir, operator_dir),
  deps=['%s/api' % operator_dir, '%s/controllers' % operator_dir, '%s/main.go' % operator_dir],
  labels=["everest-operator"]
)

# Live update the Everest operator manager without generating a new pod
load('ext://restart_process', 'docker_build_with_restart')
docker_build_with_restart('perconalab/everest-operator',
  context='%s' % operator_dir,
  dockerfile='./everest-operator.Dockerfile',
  entrypoint='/home/tilt/manager',
  only=['%s/bin/manager' % operator_dir],
  live_update=[
    sync('%s/bin/manager' % operator_dir, '/home/tilt/manager'),
  ]
)

# Apply Everest operator manifests
k8s_yaml(namespace_inject(kustomize('%s/config/default' % operator_dir, kustomize_bin='%s/bin/kustomize' % operator_dir), namespace))
k8s_resource(
  workload='everest-operator-controller-manager',
  objects=[
  	'everest-operator-system:namespace',
    'backupstorages.everest.percona.com:customresourcedefinition',
    'databaseclusterbackups.everest.percona.com:customresourcedefinition',
    'databaseclusterrestores.everest.percona.com:customresourcedefinition',
    'databaseclusters.everest.percona.com:customresourcedefinition',
    'databaseengines.everest.percona.com:customresourcedefinition',
    'monitoringconfigs.everest.percona.com:customresourcedefinition',
    'everest-operator-controller-manager:serviceaccount',
    'everest-operator-leader-election-role:role',
    'everest-operator-manager-role:clusterrole',
    'everest-operator-metrics-reader:clusterrole',
    'everest-operator-proxy-role:clusterrole',
    'everest-operator-leader-election-rolebinding:rolebinding',
    'everest-operator-manager-rolebinding:clusterrolebinding',
    'everest-operator-proxy-rolebinding:clusterrolebinding',
  ],
  resource_deps = [
  	'namespace',
	'pxc-operator',
	'psmdb-operator',
	'pg-operator',
  ],
  new_name='everest-operator',
  labels=["everest-operator"]
)

#################################
############ Everest ############
#################################

# Build backend
local_resource(
  'backend-build',
  'CGO_ENABLED=0 GOOS=linux GOARCH=amd64 make -C %s build' % backend_dir,
  deps=['%s/api' % backend_dir, '%s/cmd' % backend_dir, '%s/pkg' % backend_dir, '%s/public' % backend_dir],
  labels=["everest"]
)

# Build frontend
local_resource(
  'frontend-build',
  'make -C %s build EVEREST_OUT_DIR=%s/public/dist' % (frontend_dir, backend_dir),
  deps=['%s/apps' % frontend_dir],
  ignore=['%s/apps/*/dist' % frontend_dir, '%s/apps/*/.turbo' % frontend_dir],
  labels=["everest"]
)

# Live update the Everest container without generating a new pod
docker_build_with_restart('perconalab/everest',
  context='%s' % backend_dir,
  dockerfile='./everest.Dockerfile',
  entrypoint='/home/tilt/everest-api',
  only=['%s/bin/percona-everest-backend' % backend_dir],
  live_update=[
    sync('%s/bin/percona-everest-backend' % backend_dir, '/home/tilt/everest-api'),
  ]
)

k8s_yaml(namespace_inject('%s/deploy/quickstart-k8s.yaml' % backend_dir, namespace))
k8s_resource(
  workload='percona-everest',
  objects=[
    'everest-admin:serviceaccount',
    'everest-admin-role:role',
    'everest-admin-role-binding:rolebinding',
	'everest-admin-cluster-role:clusterrole',
	'everest-admin-cluster-role-binding:clusterrolebinding',
    'everest-admin-token:secret',
  ],
  new_name='everest',
  port_forwards=8080,
  labels=['everest'],
)
