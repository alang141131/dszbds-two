[Uploading admin.html…]()
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>虚拟购物 - 数据统计</title>
  <script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body { font-family: Arial, sans-serif; background: #f5f5f5; padding: 20px; }
    .container { max-width: 800px; margin: 0 auto; background: #fff; padding: 20px; border-radius: 8px; box-shadow: 0 2px 4px rgba(0,0,0,0.1); }
    h1 { font-size: 20px; color: #333; margin-bottom: 20px; text-align: center; }
    .btn { padding: 10px 20px; border: none; border-radius: 4px; background: #457b9d; color: #fff; font-size: 16px; cursor: pointer; margin-bottom: 20px; margin-right: 10px; }
    .btn.export { background: #e63946; }
    .stats-table { width: 100%; border-collapse: collapse; margin-bottom: 20px; }
    .stats-table th, .stats-table td { padding: 10px; text-align: left; border-bottom: 1px solid #eee; }
    .stats-table th { background: #f9f9f9; }
    .loading { text-align: center; padding: 20px; color: #666; }
    .toast { position: fixed; top: 20px; left: 50%; transform: translateX(-50%); background: rgba(0,0,0,0.7); color: #fff; padding: 10px 20px; border-radius: 4px; font-size: 14px; display: none; z-index: 999; }
    .toast.active { display: block; }
  </style>
</head>
<body>
  <div class="container">
    <h1>虚拟购物 - 数据统计</h1>
    <button class="btn" onclick="loadData()">刷新数据</button>
    <button class="btn export" onclick="exportCSV()">导出CSV数据</button>
    <div class="loading" id="loading">正在加载数据...</div>
    <table class="stats-table" id="statsTable" style="display: none;">
      <thead>
        <tr>
          <th>商品名称</th>
          <th>单价（元）</th>
          <th>销量（件）</th>
          <th>销售额（元）</th>
        </tr>
      </thead>
      <tbody id="statsBody"></tbody>
    </table>
  </div>
  <div class="toast" id="toast"></div>

  <script>
    // ========== 配置项（替换成你的Supabase信息） ==========
    const SUPABASE_URL = 'https://pewvoezygdzxnykulnzp.supabase.co'; // 和用户端一致
    const SUPABASE_SERVICE_KEY = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6InBld3ZvZXp5Z2R6eG55a3VsbnpwIiwicm9sZSI6InNlcnZpY2Vfcm9sZSIsImlhdCI6MTc3MjE3Njg5MCwiZXhwIjoyMDg3NzUyODkwfQ.xvAVU9dByE3GRTuQjOlwEQPD-5iuEOSyHOPE1yoCorY'; // 管理员密钥
    // 商品列表（和用户端完全一致）
    const PRODUCTS = [
      { id: 1, name: '福鼎白茶-白毫银针', price: 1500 },
      { id: 2, name: '福鼎白茶-白牡丹', price: 1200 },
      { id: 3, name: '福鼎白茶-寿眉', price: 800 },
      { id: 4, name: '福鼎白茶-贡眉', price: 1000 },
      { id: 5, name: '福鼎白茶-老白茶', price: 2000 },
      { id: 6, name: '福鼎白茶-荒野银针', price: 2500 }
    ];

    // ========== 全局变量（仅声明一次！） ==========
    let supabaseInstance; // 改名避免重复声明

    // ========== 初始化 ==========
    async function init() {
      try {
        // 初始化Supabase（仅一次）
        if (window.supabase) {
          supabaseInstance = window.supabase.createClient(SUPABASE_URL, SUPABASE_SERVICE_KEY);
        } else {
          throw new Error('Supabase库未加载');
        }
        // 加载数据
        await loadData();
      } catch (err) {
        showToast('初始化失败，无法加载数据');
        console.error('初始化错误：', err);
        document.getElementById('loading').textContent = '加载失败，请检查配置';
      }
    }

    // ========== 数据统计 ==========
   async function loadData() {
  document.getElementById('loading').style.display = 'block';
  document.getElementById('statsTable').style.display = 'none';
  
  try {
    // 1. 检查Supabase是否初始化
    if (!supabaseInstance) {
      throw new Error('Supabase未初始化，请核对URL/密钥');
    }

    // 2. 读取orders表所有数据（不限制字段，避免漏读）
    const { data: orders, error } = await supabaseInstance.from('orders').select('*');
    if (error) throw new Error('读取数据失败：' + error.message);

    // 3. 初始化销量统计（所有商品销量默认0）
    const salesStats = PRODUCTS.reduce((obj, p) => {
      obj[p.id] = { name: p.name, price: p.price, count: 0, sales: 0 };
      return obj;
    }, {});

    // 4. 统计数据（核心：正确解析文本格式的商品ID）
    if (orders && Array.isArray(orders) && orders.length > 0) {
      orders.forEach(order => {
        // 把文本转成商品ID数组（兼容空数据）
        let productIds = [];
        if (order.products && typeof order.products === 'string') {
          try {
            productIds = JSON.parse(order.products); // 文本转数组
          } catch (e) {
            // 如果解析失败，手动拆分（比如"1,2,3"转成[1,2,3]）
            productIds = order.products.split(',').map(id => parseInt(id)).filter(id => !isNaN(id));
          }
        }

        // 统计销量和销售额
        productIds.forEach(productId => {
          if (salesStats[productId]) {
            salesStats[productId].count += 1;
            salesStats[productId].sales += salesStats[productId].price;
          }
        });
      });
      console.log('统计到的订单数：', orders.length); // 控制台看是否读到订单
    } else {
      showToast('数据库里暂无订单数据');
    }

    // 5. 渲染表格
    const tbody = document.getElementById('statsBody');
    tbody.innerHTML = '';
    Object.values(salesStats).forEach(stat => {
      tbody.innerHTML += `
        <tr>
          <td>${stat.name}</td>
          <td>¥${stat.price}</td>
          <td>${stat.count}</td>
          <td>¥${stat.sales}</td>
        </tr>
      `;
    });

    // 6. 显示表格，隐藏加载
    document.getElementById('loading').style.display = 'none';
    document.getElementById('statsTable').style.display = 'table';
    showToast('数据加载成功！共统计 ' + (orders?.length || 0) + ' 条订单');
  } catch (err) {
    showToast('数据加载失败：' + err.message.substring(0, 50));
    console.error('管理端读取数据错误：', err);
    document.getElementById('loading').textContent = '加载失败：' + err.message;
  }
}
           // ========== 导出CSV ==========
    function exportCSV() {
      // 检查表格是否显示
      const statsTable = document.getElementById('statsTable');
      if (statsTable.style.display === 'none') {
        showToast('请先加载数据再导出');
        return;
      }

      // 组装CSV内容
      let csvContent = '商品名称,单价（元）,销量（件）,销售额（元）\n';
      const rows = statsTable.querySelectorAll('tbody tr');
      
      rows.forEach(row => {
        const cells = row.querySelectorAll('td');
        const name = cells[0].textContent;
        const price = cells[1].textContent.replace('¥', '');
        const count = cells[2].textContent;
        const sales = cells[3].textContent.replace('¥', '');
        csvContent += `${name},${price},${count},${sales}\n`;
      });

      // 下载CSV文件
      const blob = new Blob([csvContent], { type: 'text/csv;charset=utf-8;' });
      const url = URL.createObjectURL(blob);
      const link = document.createElement('a');
      link.setAttribute('href', url);
      link.setAttribute('download', `虚拟购物数据_${new Date().toLocaleDateString()}.csv`);
      link.style.display = 'none';
      document.body.appendChild(link);
      link.click();
      document.body.removeChild(link);
      showToast('CSV导出成功');
    }

    // ========== 工具函数 ==========
    function showToast(text) {
      const toast = document.getElementById('toast');
      toast.textContent = text;
      toast.classList.add('active');
      setTimeout(() => toast.classList.remove('active'), 2000);
    }

    // 页面加载完成后初始化
    document.addEventListener('DOMContentLoaded', init);
  </script>
</body>
</html>
