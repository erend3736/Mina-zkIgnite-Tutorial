sudo su
cd
sleep(2)

ufw allow 22 && ufw allow 3000
ufw enable
sleep(1)

sudo apt-get update && sudo apt-get upgrade -y
sleep(1)

sudo apt install curl git build-essential 
sleep(1)

sudo apt install yarn
sleep(1)

sudo apt-get install nodejs
sleep(1)

sudo apt install npm
sleep(2)

nodejs -v
npm -v

curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -

sudo apt-get install nodejs

node -v

npm -v

git clone https://github.com/o1-labs/zkapp-cli

npm instal -g zkapp-cli@0.5.3

zk --version

npm install -g zkapp-cli

git config --global user.email (mail adresiniz)

git config --global user.name (kullanıcı ismi)


zk project 04-zkapp-browser-ui --ui next

cd 04-zkapp-browser-ui

git remote add origin oluşturduğumuz repo'nun url adresi

git push -u origin main

contrat çalıştırıyoruz
  
cd
cd 04-zkapp-browser-ui/contracts/

npm run build

cd
cd 04-zkapp-browser-ui/ui/pages

sudo nano zkappWorker.ts

import {
    Mina,
    isReady,
    PublicKey,
    PrivateKey,
    Field,
    fetchAccount,
} from 'snarkyjs'

type Transaction = Awaited<ReturnType<typeof Mina.transaction>>;

// ---------------------------------------------------------------------------------------

import type { Add } from '../../contracts/src/Add';

const state = {
    Add: null as null | typeof Add,
    zkapp: null as null | Add,
    transaction: null as null | Transaction,
}

// ---------------------------------------------------------------------------------------

const functions = {
    loadSnarkyJS: async (args: {}) => {
        await isReady;
    },
    setActiveInstanceToBerkeley: async (args: {}) => {
        const Berkeley = Mina.BerkeleyQANet(
            "https://proxy.berkeley.minaexplorer.com/graphql"
        );
        Mina.setActiveInstance(Berkeley);
    },
    loadContract: async (args: {}) => {
        const { Add } = await import('../../contracts/build/src/Add.js');
        state.Add = Add;
    },
    compileContract: async (args: {}) => {
        await state.Add!.compile();
    },
    fetchAccount: async (args: { publicKey58: string }) => {
        const publicKey = PublicKey.fromBase58(args.publicKey58);
        return await fetchAccount({ publicKey });
    },
    initZkappInstance: async (args: { publicKey58: string }) => {
        const publicKey = PublicKey.fromBase58(args.publicKey58);
        state.zkapp = new state.Add!(publicKey);
    },
    getNum: async (args: {}) => {
        const currentNum = await state.zkapp!.num.get();
        return JSON.stringify(currentNum.toJSON());
    },
    createUpdateTransaction: async (args: {}) => {
        const transaction = await Mina.transaction(() => {
            state.zkapp!.update();
        }
        );
        state.transaction = transaction;
    },
    proveUpdateTransaction: async (args: {}) => {
        await state.transaction!.prove();
    },
    getTransactionJSON: async (args: {}) => {
        return state.transaction!.toJSON();
    },
};

// ---------------------------------------------------------------------------------------

export type WorkerFunctions = keyof typeof functions;

export type ZkappWorkerRequest = {
    id: number,
    fn: WorkerFunctions,
    args: any
}

export type ZkappWorkerReponse = {
    id: number,
    data: any
}
if (process.browser) {
    addEventListener('message', async (event: MessageEvent<ZkappWorkerRequest>) => {
        const returnData = await functions[event.data.fn](event.data.args);

        const message: ZkappWorkerReponse = {
            id: event.data.id,
            data: returnData,
        }
        postMessage(message)
    });
}
 ```
 ctrl x ile çıkıyoruz y yazıp enter diyoruz.
 
 2. dosyayı düzenliyoruz
  ```
sudo nano zkappWorkerClient.ts
  ```
 ```
 import {
    fetchAccount,
    PublicKey,
    PrivateKey,
    Field,
} from 'snarkyjs'

import type { ZkappWorkerRequest, ZkappWorkerReponse, WorkerFunctions } from './zkappWorker';

export default class ZkappWorkerClient {

    // ---------------------------------------------------------------------------------------

    loadSnarkyJS() {
        return this._call('loadSnarkyJS', {});
    }

    setActiveInstanceToBerkeley() {
        return this._call('setActiveInstanceToBerkeley', {});
    }

    loadContract() {
        return this._call('loadContract', {});
    }

    compileContract() {
        return this._call('compileContract', {});
    }

    fetchAccount({ publicKey }: { publicKey: PublicKey }): ReturnType<typeof fetchAccount> {
        const result = this._call('fetchAccount', { publicKey58: publicKey.toBase58() });
        return (result as ReturnType<typeof fetchAccount>);
    }

    initZkappInstance(publicKey: PublicKey) {
        return this._call('initZkappInstance', { publicKey58: publicKey.toBase58() });
    }

    async getNum(): Promise<Field> {
        const result = await this._call('getNum', {});
        return Field.fromJSON(JSON.parse(result as string));
    }

    createUpdateTransaction() {
        return this._call('createUpdateTransaction', {});
    }

    proveUpdateTransaction() {
        return this._call('proveUpdateTransaction', {});
    }

    async getTransactionJSON() {
        const result = await this._call('getTransactionJSON', {});
        return result;
    }

    // ---------------------------------------------------------------------------------------

    worker: Worker;

    promises: { [id: number]: { resolve: (res: any) => void, reject: (err: any) => void } };

    nextId: number;

    constructor() {
        this.worker = new Worker(new URL('./zkappWorker.ts', import.meta.url))
        this.promises = {};
        this.nextId = 0;

        this.worker.onmessage = (event: MessageEvent<ZkappWorkerReponse>) => {
            this.promises[event.data.id].resolve(event.data.data);
            delete this.promises[event.data.id];
        };
    }

    _call(fn: WorkerFunctions, args: any) {
        return new Promise((resolve, reject) => {
            this.promises[this.nextId] = { resolve, reject }

            const message: ZkappWorkerRequest = {
                id: this.nextId,
                fn,
                args,
            };

            this.worker.postMessage(message);

            this.nextId++;
        });
    }
}
 
 

  

  
  