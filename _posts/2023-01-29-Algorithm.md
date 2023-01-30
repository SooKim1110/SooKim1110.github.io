---
title: 알고리즘 개념 정리
date: 2023-01-29 10:00:00 +0900
categories: [Archive]
tags: [Algorithm]
---

# 알고리즘 정리
<br/>

# 1) 알고리즘 기초
## DFS
``` cpp
#define MAX 1001;

bool visited[MAX];
vector<int> adj[MAX];

void dfs(int x){
  visited[x] = 1;
  for (int n: adj[x]){
    if (!visited[n]){
      dfs(n);
    }
  }
}

int main(){
  ...
  for (int i = 0; i<n; i++){
    int s,e;
    cin >> s >> e;
    adj[s].push_back(e);
    adj[e].push_back(s);
  }
  ...
}
```



## BFS
``` cpp
#define MAX 1001;

bool visited[MAX];
vector<int> adj[MAX];
 
void bfs(int x){
  queue<int> que;
  visited[x] = 1;
  que.push(x);

  int cur;
  while (!que.empty()){
    int cur = que.front(); que.pop();
    for (int n: adj[cur]){
      if (!visited[n]){
        visited[n] = 1;
        que.push(n);
      }
    }
  }
  
}

int main(){
  ...
  for (int i = 0; i<n; i++){
    int s,e;
    cin >> s >> e;
    adj[s].push_back(e);
    adj[e].push_back(s);
  }
  ...
}
``` 

## 이분탐색(Binary Search)

``` cpp
#define MAX 1001;

// 최솟값 찾기
int BinarySearch(int n){
  int low = 0, high = MAX, mid;
  while (low <= high){
    mid = (low+high)/2;
    if (check(mid)) high = mid-1;
    else low = mid+1;
  }
  return high + 1;
}

// 최댓값 찾기
int BinarySearch(int n){
  int low = 0, high = MAX, mid;
  while (low <= high){
    mid = (low+high)/2;
    if (check(mid)) low = mid+1;
    else high = mid -1;
  }
  return low-1;
}
``` 

## 머지 소트(Merge Sort)
``` cpp
#define MAX 1001

int num[MAX], buf[MAX];

// from부터 to 앞부분까지 정렬
void mergeSort(int from, int to){
  if (from >= to-1) return;
  int mid = (from +to)/2;
  mergeSort(from, mid);
  mergeSort(mid,to);
  merge(from, mid, to);
}

void merge(int from, int mid, int to){
  int i1 = from, i2 = mid, i = from;
  while (i1 < mid && i2 < to){
    if (num[i2] < num[i]){
      buf[i++] = num[i2++];
    }
    else {
      buf[i++] = num[i1++];
    }
  }

  //배열의 남은 숫자들 채워넣기
  while (i1<mid){
    buf[i++] = num[i1++];
  }
  while (i2 <to){
    buf[i++] = num[i2++];
  }
  // 실제 배열로 buf 숫자들 옮기기
  for (i = from; i< to; ++i)
    num[i] = buf[i];
}
``` 

## 백트래킹(Backtracking)
``` cpp
// 예시)
int arr[10], ans[10];
bool visited[10];

void backtracking(int len) {
	if (len == M) {
		// 종료시점에 나와야하는 결과 
		return;
	}

	for (i = 0; i < arr.size(); ++i) {
		if (!check(i)) {
			ans[len] = arr[i];
			visited[i] = 1;
			backtracking(len + 1);
			visited[i] = 0;
		}
	}
}
```
```
#N-queen 크기가 N인 일차원 배열, 각 열에 몇번째 행에 퀸이 있는지를 저장 
#include <iostream>
#include <vector>
#include <algorithm>
#include <cstdlib>
#include <queue>
using namespace std;
typedef long long ll;

int n;
int ans = 0;
int board[16];


void dfs(int row){
  if (row == n+1){
    ans++;
    return;
  }
  
  for (int i = 1; i<=n; i++){
    bool possible = true;
    // 지금까지 놓았던 퀸 위치 확인
    for (int j = 1; j< row; j++){
      if (board[j] == i || abs(i - board[j]) == row - j){
        possible = false;
        break;
      }
    }
    if (possible){
      board[row] = i;
      dfs(row+1);
    }
  }
}

int main()
{
  ios::sync_with_stdio(0);
  cin.tie(0);

  cin >> n;

  for (int i = 1; i<=n; i++){
    board[1] = i;
    dfs(2);
  }  
  cout << ans;
}
```

## 다이나믹 프로그래밍 (DP)
``` cpp
// 예시
#define MAX		1000

int num[MAX];
int dp[MAX];

//top down
int DP(int k) {
	//	기저조건
	if (k == 0) return 0;
	//	Memoization
	if (dp[k] != -1) return dp[k];
	//	점화식
	dp[k] = max(DP(k-1), DP(k-2));
	
	return dp[k];
}

//bottom up
int DP(int k) {
  dp[0] = 1;
  dp[1] = 1;
  for (int i = 2; i<=k; ++i){
    dp[k] = max(dp[k-1] + dp[k-2]);
  }
	return dp[k];
}

``` 
<br/>

# 2) 자료구조
- Stack, Queue, Heap(Priority Queue), Map, Set (c++ stl)

## 인덱스 트리
- 구간 조회, 단일 업데이트
``` cpp
#define MAX         1000000
#define SZ_TR       2097152

int OFFSET, tree[SZ_TR];
int num[MAX+1];

// tree[0] 사용하지 않음, 리프 인덱스는 0 부터 시작, 왼쪽 노드(짝수) 오른쪽 노드(홀수)
void init(int N){
  int i;
  //OFFSET 계산 - 리프 노드 시작 위치
  for (OFFSET = 1; OFFSET < N; OFFSET *= 2);
  // 리프 노드에 수 채우기
  for (i = 0; i<N; ++i) tree[OFFSET + i] = num[i];
  // 남은 노드는 디폴트 값으로 채우기
  for (i = N; i<OFFSET; ++i) tree[OFFSET + i] = 0;
  // 구간합 계산
  for (i = OFFSET -1; i > 0; --i) tree[i] = tree[i*2] + tree[i*2+1];
}

int query(int from, int to){
  int res = 0;
  from += OFFSET, to += OFFSET;
  
  while (from <= to){
    if (from %2 == 1) res += tree[from++];
    if (to %2 == 0)  res += tree[to--];
    from /= 2; to /=2;
  }
  return res;
}

void update(int idx, int val){
  // idx 노드 업데이트(리프 인덱스는 0부터 시작!)
  idx += OFFSET;
  tree[idx] = val;

  idx /= 2;

  // 부모 노드들 업데이트
  while (idx >0){
    tree[idx] = tree[idx*2] + tree[idx*2+1];
    idx /= 2;
  }
}

int findKth(int kth){
  int idx = 1, left, right;

  while (idx < OFFSET){
    left = idx*2, right = left + 1;
    if (tree[left] >= kth) idx = left;
    else kth -= tree[left], idx = right;
  }
  return idx - OFFSET;
}
``` 

## 트라이(Trie)
- 문자열 검색
``` cpp
struct Trie {
  // 포함하는 문자 종류 개수만큼 배열 생성(다음으로 가리키는 문자) 
  Trie* Node[26];
  // 이 노드에서 끝나는 문자열이 있는가
  bool fin;

  Trie()
  {
    fin = false;
    for (int i = 0; i < 26; i++)
      Node[i] = NULL;
  }

  void Insert(string s){
    Trie* pNode = this;
    for (int i = 0; i<s.length(); ++i){
      int idx = s[i] - 'A';
      if (!pNode -> Node[idx])
        pNode -> Node[idx] = new Trie();
      pNode = pNode->Node[idx];
    }
    pNode -> fin = true;
  }

  void Search(string s){
    Trie* pNode = this;
    for (int i = 0; i<s.length(); ++i){
      int idx = s[i] - 'A';
      if (!pNode -> Node[idx])
        return false;
      pNode = pNode->Node[idx];
    }
    return (!pNode && pNode->fin);
	}
};

  int main(){
    Trie* root = new Trie();
    for (int i = 0; i<N; i++){
      root->Insert(words[i]);
    }
  }
``` 

<br/>

# 3) 정수론
## 유클리드 호제법(최대공약수 구하기)
``` cpp
typedef long long ll;

ll gcd(ll a, ll b){
  if (b == 0) return a;
  else return gcd(b,a%b);
}

ll gcd2(ll a, ll b) {
	ll t;

	while (b) {
		t = a % b;
		a = b;
		b = t;
	}

	return a;
}
``` 

## 확장 유클리드 호제법
- 정수 m,n 이 있을 때, am + bn = gcd(m,n)의 해가 되는 정수 a, b 짝 찾아내기
- 밑의 코드에서 a,b가 1일 때 주의
``` cpp
typedef long long ll;

ll ee(int a, int b, int K) {
	ll s, s0 = 1, s1 = 0, t, t0 = 0, t1 = 1, q, r;

	while (b != 0)
  {
    q = a / b;
    r = a % b;
    s = s0 - s1 * q;
    t = t0 - t1 * q;

    // 다음 계산
    a = b;
    b = r;
    s0 = s1, s1 = s;
    t0 = t1, t1 = t;
  }

	//음수라면 가능한 양수로 바꾸어줌
  t0 = (t0 % K + K) % K;
  return t0;
}
``` 

## 에라스토테네스의 체 (소수 구하기)
``` cpp
#define MAX    	1001

bool flag[MAX+1];
vector<int> primes;

int main(){
  for (int i = 2; i<= MAX; ++i){
    if (!flag[i]){
      primes.push_back(i);
      for (ll j = (ll) i*i; j <= MAX; j += i){
        flag[j] = 1;
      }
    }
  }
}
```  

<br/>

# 4) 조합론
## 조합 (Combination)
- 파스칼 삼각형 성질 이용하여 조합 경우의 수 계산
``` cpp
#define MAX    	1001
#define DIV			10007

int combi[MAX + 1][MAX + 1];

int main() {
  ... 
	for (int n = 0; n <= N; ++n) {
		combi[n][0] = combi[n][n] = 1;
		for (int k = 1; k < n; ++k) 
      combi[n][k] = (combi[n - 1][k - 1] + combi[n - 1][k]) % DIV;
	}
  ...
}

``` 
## 순열, 조합 구현
순열 (순서 따지고, 중복 허용 안함)
- check 배열로 사용한 숫자인지 확인
``` cpp
int K,N;
bool check[100];
vector<int> chosen;

void permutation(int cnt){
    if (cnt == K) {
        for (int x: chosen) cout << x << " ";
        cout << endl;
        return;
    }
    
    //이번 인덱스 선택함
    for (int i = 1; i<=N; i++){
        if (!check[i]){
            check[i] = 1;
            chosen.push_back(i);
            permutation(cnt+1);
            chosen.pop_back();
            check[i] = 0;
        }
    }
    
}

int main(){
    N = 4;
    K = 3;
    permutation(0);
}
```

조합 (순서 따지지 않고, 중복 허용 안함)
- 반복문 시작 값이 이전에 선택한 값 +1 부터
``` cpp
int K,N;
vector<int> chosen;

void combination(int idx, int cnt){
    if (cnt == K) {
        for (int x: chosen) cout << x << " ";
        cout << endl;
        return;
    }
    
    for (int i = idx; i<=N; i++){
        chosen.push_back(i);
        combination(i+1,cnt+1);
        chosen.pop_back();
    }
    
}

int main(){
    N = 4;
    K = 3;
    combination(1,0);
}
```


<br/>

# 5) 그래프

## 유니온 파인드
- 서로소 집합 (교집합이 공집합인 집합)
``` cpp
struct UnionFind
{
  vector<int> parent, rank;

  UnionFind(int N) : parent(N + 1), rank(N + 1, 1)
  {
    for (int i = 1; i <= N; i++)
    {
      parent[i] = i;
    }
  }

  // 루트 노드 찾기, parent 배열 업데이트 
  int find(int a)
  {
    if (a == parent[a])
      return parent[a];
    return parent[a] = find(parent[a]);
  }

  void merge(int a, int b)
  {
    int u = find(a);
    int v = find(b);
    if (u == v) return;

    // v의 트리 높이가 더 높도록
    if (rank[u] > rank[v])
    {
      swap(u,v);
    }
    //트리 합치기
    parent[u] = v;

    if (rank[u] == rank[v])
      ++rank[v];
  }
};

int main()
{
  ios::sync_with_stdio(0);
  cin.tie(0);

  int n, m;
  cin >> n >> m;

  UnionFind uf = UnionFind(n);
  int a, b, c;

  for (int i = 0; i < m; ++i)
  {
    cin >> a >> b >> c;
    if (a == 0)
    {
      uf.merge(b, c);
    }
    else
    {
      if (uf.find(b) == uf.find(c)) cout << "YES\n";
      else cout << "NO\n";
    }
  }
}
``` 

## 최소신장 트리 (MST)
- 크루스칼 알고리즘: 간선 오름차순 정렬, 사이클이 없으면 간선 선택
- 구현에는 Union Find 활용. 최상위 부모가 같으면 사이클
``` cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <queue>

using namespace std;

typedef long long ll;

struct UnionFind{
  vector<int> parent, rank;

  UnionFind(int N): parent(N+1), rank(N+1,1){
    for (int i = 1; i<=N; ++i){
      parent[i] = i;
    }
  }

  int find(int a){
    if (parent[a] == a) return a;
    else return parent[a] = find(parent[a]);
  }

  void merge(int a, int b){
    int u = parent[a];
    int v = parent[b];

    if (u == v) return;

    // v의 트리 높이가 더 높도록
    if (rank[u] > rank[v]) swap(u,v);
    parent[u] = v;
    if (rank[u] == rank[v]){
      ++rank[v];
    }
  }
};

vector<pair<int, pair<int, int>>> adjs;

int main()
{
  ios::sync_with_stdio(0);
  cin.tie(0);

  int n, m, a, b, c;
  cin >> n >> m;

  for (int i = 0; i < m; i++)
  {
    cin >> a >> b >> c;
    adjs.push_back({c, {a, b}});
  }

  //간선 가중치로 오름차순 정렬
  sort(adjs.begin(), adjs.end());

  int cnt = 0, ans = 0;
  UnionFind uf = UnionFind(n);
  //간선 선택하기
  for (int i = 0; i<m; i++){
    int a = adjs[i].second.first;
    int b = adjs[i].second.second;
    if (!(uf.find(a) == uf.find(b))){
      ans += adjs[i].first;
      uf.merge(a,b);
      cnt++;
    }

    if (cnt == n-1) break;
  }
  cout << ans << '\n';
}
``` 
## 위상 정렬
- 비순환 방향 그래프에서 그래프의 방향성을 거스르지 않고 정점 나열
- DFS 역순을 구하거나 자신에게 들어오는 간선의 개수(indegree)를 활용
- 모든 원소 방문하기 전 큐가 비면 사이클이 존재하는 것 (dfs사용할 때는 finish 배열 만들어서 끝나면 저장. visited되었는데 finish 아니면 사이클)
``` cpp
queue<int> que;
int degree[32001];
vector<int> adjs[32001];

int main()
{
  ios::sync_with_stdio(0);
  cin.tie(0);

  int n, m, a,b;
  cin >> n >> m;

  for (int i = 0; i < m; i++)
  {
    cin >> a >> b;
    degree[b]++;
    adjs[a].push_back(b);
  }
  // indegree가 0 인 정점들 큐에 넣어주기
  for (int i = 1; i<=n; i++){
    if (degree[i] == 0){
      que.push(i);
    }
  }
  // indegree 지워가면서 0이 될 때마다 큐에 넣기
  while (!que.empty()){
    int front = que.front();
    que.pop();
    cout << front << " ";
    for (int x: adjs[front]){
      if (degree[x] >0) degree[x]--;
      if (degree[x] == 0) que.push(x);
    }
  }
}
```
```  
void dfs(int here) {
    visited[here] = true;
    for (auto there : vt[here]) {
        if (!visited[there])
            dfs(there);
        else if (!finish[there])
            cycle = 1;
    }
    finish[here] = true;
    st.push(here);
}
``` 

## LCA (최소 공통 조상)
- 2^k 번째 조상을 배열에 저장하여 시간복잡도 O(logN)으로 줄이기
``` cpp
#include <iostream>
#include <vector>
#include <algorithm>

using namespace std;

#define MAX         100001
#define MAX_DEPTH   20

int n;
vector<int> adjs[MAX];
int depth[MAX];
bool visit[MAX];
int parent[MAX][MAX_DEPTH+1];

void dfs(int now, int len){
  visit[now] = 1;
  depth[now] = len;

  for (int next:adjs[now]){
    if (visit[next]) continue;
      parent[next][0] = now;
      dfs(next, len+1);
    
  }
}

void dp(){
   for(int j=1;j<=MAX_DEPTH;j++){      
      for(int i=1;i<=n;i++){
        parent[i][j]=parent[parent[i][j-1]][j-1];
    }
  }
}

int lca(int x, int y){
  if (depth[x] > depth[y]) swap(x,y);
  //같은 높이가 되도록 낮은쪽을 올려줌
  for (int i = MAX_DEPTH; i>=0; i--){
    if (depth[parent[y][i]] >= depth[x]) y = parent[y][i];
  }
  if (x == y) return x;

  // 둘의 부모가 같아지기 직전까지 올려줌
  for (int i = MAX_DEPTH; i>=0; i--){
    if (parent[x][i]!= parent[y][i]){
      x = parent[x][i];
      y = parent[y][i];
    }
  }
  return parent[x][0];
}


int main()
{
  ios::sync_with_stdio(0);
  cin.tie(0);

  cin >> n;

  for (int i = 0; i<n-1; i++){
    int a,b;
    cin >> a >> b;
    adjs[a].push_back(b);
    adjs[b].push_back(a);
  }
  depth[0] = -1;
  //각 노드들의 깊이 depth에 저장
  dfs(1,0);

  //parent 배열 채우기
  dp();

  int m;
  cin >> m;

  for (int i = 0; i<m; i++){
    int a,b;
    cin >> a >> b;
    cout << lca(a,b) << '\n';
  }
}
``` 

## 다익스트라
``` cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <queue>

using namespace std;

#define MAX_V   20001

vector<pair<int, int> > adj[MAX_V];
int dist[MAX_V], visit[MAX_V];

void dijkstra(int s){
  priority_queue<pair<int, int> > pq;
  pq.push({0,s});
  dist[s] = 0;

  while (!pq.empty()){
    int n = pq.top().second;
    int w = -pq.top().first;
    pq.pop();

    if (visit[n]) continue;
    
    visit[n] = 1;
    for (auto p: adj[n]){
      int x = p.second;
      int w2 = -p.first;
      if (dist[x] < w + w2) continue;
      dist[x] = w+w2;
      pq.push({-dist[x], x});
    }
  }

  
}
int main()
{
  ios::sync_with_stdio(0);
  cin.tie(0);

  int v,e, s;
  cin >> v >> e >> s; 

  for (int i = 0; i<e; i++){
    int a,b,w;
    cin >> a >> b >> w;
    adj[a].push_back({-w,b});
  }

  for (int i = 1; i<=v; i++) dist[i] =  100000000;
  dijkstra(s);
  
  for (int i = 1; i<=v; i++){
    if ( dist[i] !=  100000000) {
       cout << dist[i] << '\n';
    }
    else cout << "INF" << '\n';
  }

}
``` 

## 벨만포드
``` cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <queue>
#include <cstring>

using namespace std;
typedef long long ll;

#define MAX_V   501
#define INF     1000000000

struct Edge{
  int from, to, cost;
  Edge(int from, int to, int cost):from(from),to(to),cost(cost){}
};

vector <Edge> edgeList;
ll dist[MAX_V];

int main()
{
  ios::sync_with_stdio(0);
  cin.tie(0);

  int n,m;
  cin >> n >> m;

  for (int i = 0; i<m; i++){
    int from, to, cost;
    cin >> from >> to >> cost;
    edgeList.push_back(Edge(from,to,cost));
  }

  for (int i = 1; i<=n; i++){
    dist[i] = INF;
  }

  dist[1] = 0;

  bool isCycle = 0;
  for (int i = 1; i<=n; i++){
    for (Edge edge: edgeList){
      if (dist[edge.from] == INF) continue;
      if (dist[edge.to] > dist[edge.from] + edge.cost){
        dist[edge.to] = dist[edge.from] + edge.cost;
        if (i == n) isCycle = 1;
      }
    }
  }

  if (isCycle) cout << "-1\n";
  else{
    for (int i = 2; i<=n; i++){
      if (dist[i] == INF) cout << "-1\n";
      else cout << dist[i] << '\n';
    }
  }
}
``` 

## 플로이드 워셜
``` cpp
#include <iostream>
#include <vector>
#include <algorithm>
#include <cstdlib>
#include <queue>
using namespace std;
typedef long long ll;


#define MAX_V     101
#define MAX_E     100001
#define INF       1000000000

int dist[MAX_V][MAX_V];

int main()
{
  ios::sync_with_stdio(0);
  cin.tie(0);

  int n,m;
  cin >> n >> m;
  
  for (int i = 1; i<=n; i++){
    for (int j = 1; j<=n; j++){
      if (i == j) dist[i][j] = 0;
      else dist[i][j] = INF;
    }
  }

  for (int i = 0; i<m; i++){
    int a,b,c;
    cin >> a >> b >> c;
    dist[a][b] = min(dist[a][b],c);
  }

  for (int k = 1; k<=n; k++){
    for (int i = 1; i<=n; i++){
      for (int j = 1; j<=n; j++){
        if (dist[i][j] > dist[i][k] + dist[k][j])
          dist[i][j] = dist[i][k] + dist[k][j];
      }
    }
  }

  for (int i = 1; i<=n; i++){
    for (int j = 1; j<=n; j++){
      if (dist[i][j] == INF) cout << "0 ";
      else cout << dist[i][j] << " ";
    }
    cout << '\n';
  }
}
```
## Python 문자열 다루기
str.isalnum() - 문자나 숫자인지 검사
str.isalpha()
str.isdigit()
str.islower()
str.isupper()

str.lower()
str.upper()

str.find() - substring이 처음 등장하는 위치의 인덱스를 리턴, 없으면 -1 리턴
str.count() - 해당 문자열이 나온 개수 리턴

str.strip() - 전달된 문자를 str 왼쪽, 오른쪽에서 제거 (인자 없으면 공백 제거)
str.lstrip()
str.rstrip()

str.split('.')

str.replace(‘old’, ‘new’, 2)

문자열 슬라이싱
str[-1] - 마지막 문자
str[::2] - 2개씩 step
str[::-1] - reverse

리스트를 문자열로 결합
" ".join(list) - 리스트 사이 공백 삽입

chr(97) - a
ord('a;) - 97

Regex
re.match('[a-z]', str) - 문자열 처음부터 매치 확인
re.search() - 문자열 전체 검색
re.findall() - 모든 문자열 list로 리턴
re.sub('[a-z]', '', str)



## 기타
int 범위: –2,147,483,648 ~ 2,147,483,647
float 범위: 3.4E-38(-3.4*10^38) ~ 3.4E+38(3.4*10^38 (7digits)
double 범위: 1.79E-308(-1.79*10^308) ~ 1.79E+308(1.79*10^308) (15digits)

ios_base::sync_with_stdio(false);
cin.tie(null);
c << '\n;

compare 함수

eg.
sort

```
bool compare(Student a, Student b) {
    return a.score < b.score;
}

int main(void) {
  sort(students, students + 5, compare);
}
```
priority queue
```
struct compare{
	bool operator()(pair<int, int>a, pair<int, int>b){
		return a.second>b.second;
	}
};

int main(){
	priority_queue<pair<int, int>, vector<pair<int, int>>, compare>pq;
	pq.push({1, 10});
	pq.push({2, 3});
	pq.push({3, 1});
	cout<<pq.top().first; // 출력 : 3
	cout<<pq.top().second; // 출력 : 1
}
```

map
[] => find, 없으면 add
insert => 이미 존재하면 추가하지 않음

```
#include<iostream>
#include<map>
#include<string>
using namespace std;
 
int main(void){
    
    map<int, string> m;
    
    m.insert(pair<int, string>(1, "hello"));
    m.insert(pair<int, string>(3, "world"));
    
    map<int, string>::iterator iter;
    
    //접근
    for(iter = m.begin(); iter != m.end(); iter++){
        cout << iter->first << ", " << iter->second << " " ;
    }
    cout << endl;
    return 0;   
}
```

stl
lower_bound()
vector<int>::iterator iter = lower_bound(v.begin(), v.end(), 22)
upper_bound()
next_permutation()
```
	do{
		for(int i=0; i<4; i++){
			cout << v[i] << " ";
		}
		cout << '\n';
	}while(next_permutation(v.begin(),v.end()));
```

재귀함수 부분집합 구하기
```
vector<int> subset;
int n = 4;
void search(int k){
  if (k == n+1){
    for (int i = 0; i<subset.size(); i++){
      cout << subset[i];
    }
  else{
    subset.push_back(k);
    search(k+1);
    subset.pop_back();
    search(k+1);
  }  
}

int main(){
  search(1);
}

```